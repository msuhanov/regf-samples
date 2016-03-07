# Flush strategies in the Windows registry
Windows implementation of the registry deals with two entities: a registry hive in memory and a registry hive on a disk. When a registry hive is mounted, it's being read or mapped into memory from a disk. When you change something in a mounted registry hive (e.g. when you create a new key or modify a value), these changes happen directly in memory. The process of writing these changes back to a disk is called a hive flush.

Before Windows 8.1 and Windows Server 2012 R2, the flush process writes modified data (also known as dirty data) to a transaction log file first (overwriting modified data from previous flush processes), then the flush process writes the same data to a primary file. If a system crash (e.g. power outage) occurs when writing to a transaction log file, a primary file will remain consistent, because no data was written to this file during the failed flush operation; if a system crash occurs when writing to a primary file, a copy of modified data from a transaction log file will be used to finish writing to a primary file, thus bringing the file back to the consistent state. Before writing to a primary file, the flush process will invalidate its header to record the inconsistent state, and after modified data was successfully stored in a primary file, the flush process will validate the header (so it's possible to tell whether a primary file is consistent or not by examining its header). The flush process for a hive is triggered by a kernel at regular time intervals or by a userspace program using the RegFlushKey() routine. This flush strategy involves writing the same data twice, thus reducing the performance of an operating system.

In Windows 8.1 and Windows Server 2012 R2, a new flush strategy was implemented: when the flush process is triggered for a specific hive (either by a kernel or by a userspace program) for the first time or after the status of a transaction log file has been reset, the header of a primary file is invalidated, and a log entry with modified (dirty) data is written to a transaction log file. When the flush process is triggered again, a new log entry with modified data is appended to a transaction log file, and a primary file remains untouched. If a system crash occurs, a transaction log file will contain log entries required to recover the consistency of a primary file and to bring it to the up-to-date state. When all users (local and remote) become inactive, or when a hive starts unloading (e.g. during the full shutdown), or when an hour has been elapsed since the latest write to a primary file, the reconcile process will write all modified data to a primary file, validate its header, and reset the status of a transaction log file (next flush processes will be overwriting old log entries with new ones). The new flush strategy has performance improvements based on the significant decrease of the number of disk writes.

However, the new flush strategy may result in data being overlooked when examining primary files of registry hives only. It's known that many forensic registry viewers are dealing with primary files exclusively.

# Experimental testing
The following experiment was conducted to verify the correctness of the findings:

1. A virtual machine with a Windows 8.1 operating system was set up.
2. This virtual machine was powered off after the installation using the following command: «shutdown /t 0 /s» (the command initiates the full shutdown, not the hybrid one).
3. The following files were copied from the virtual machine to record the *before* state (the first file in the list is the primary file, other files are transaction log files):

    * C:\Windows\System32\config\SYSTEM
    * C:\Windows\System32\config\SYSTEM.LOG
    * C:\Windows\System32\config\SYSTEM.LOG1
    * C:\Windows\System32\config\SYSTEM.LOG2

4. The virtual machine was started again.
5. Once the operating system finished booting, the Regedit program was launched, and the «testAAAA» key was created in the root of the System hive. The Regedit program was closed after this.
6. The user activity was simulated.
7. Approximately 16 minutes after the boot, the Regedit program was launched again, and the «testBBBB» key was created in the root of the System hive. The Regedit program was closed after this.
8. The user activity was simulated.
9. Approximately 45 minutes after the boot, the hybrid shutdown was initiated from the Start screen. The virtual machine was powered off.
10. The files whose paths are mentioned above were copied from the virtual machine to record the *after* state.

## Results
The following results were obtained after comparing the files copied:

1. The contents of the «SYSTEM» file did change.
2. The contents of the «SYSTEM.LOG» file didn't change (in fact, this file is empty).
3. The contents of the «SYSTEM.LOG1» file did change.
4. The contents of the «SYSTEM.LOG2» file didn't change.

The following changes were observed in the «SYSTEM» file:

1. The byte at offset 4 is 0xCE in the *before* state and 0xCF in the *after* state.
2. The byte at offset 144 is 0x00 in the *before* state and 0x01 in the *after* state.

As expected, there are no signs of the «testAAAA» and «testBBBB» keys in both states of the «SYSTEM» file. However, signs of these keys were found in the «SYSTEM.LOG1» file (the *after* state only). The contents of the «SYSTEM.LOG1» file did change cardinally between the states.

In the *before* state, the «SYSTEM.LOG1» file has 9 log entries (0xC5-0xCD). In the *after* state, the same file has 32 log entries (0xCE-0xED). The «testAAAA» key is first seen in the log entry numbered 0xD5. The «testBBBB» key is seen in the log entry numbered 0xDF. The difference between the last written timestamps of these two keys is 948 seconds (approximately 16 minutes).

These results match what was expected.

## Registry files
The *before* and *after* states of the «SYSTEM», «SYSTEM.LOG», «SYSTEM.LOG1», and «SYSTEM.LOG2» files are available for an examination on GitHub in this directory.

# Conclusions
When dealing with registry files from Windows 8.1, Windows Server 2012 R2, or Windows 10, a computer forensic examiner needs to take into account data in transaction log files. Without looking at transaction log files, an examiner may not see the latest changes happened to the registry.

___
© 2016 Maxim Suhanov

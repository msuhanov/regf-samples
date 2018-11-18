# Missing log entries for a dirty primary file
A new flush strategy implemented in Windows 8.1 and Windows Server 2012 R2 writes dirty data to a transaction log file, but not immediately to a primary file, which is written later by the reconcile process (in order to reduce the number of disk writes). When the dual-logging scheme is used, a primary file has two corresponding transaction log files (*.LOG1 and *.LOG2) used to store dirty data (this dirty data becomes unreconciled when stored in a transaction log file). A kernel will rotate transaction log files when necessary (for example, when a current transaction log file becomes too large).

Under certain conditions, a kernel will rotate the transaction log file twice before a primary file is reconciled *in full*. In this situation, a primary file should be reconciled nevertheless, because old (but not yet applied) log entries are going to be overwritten with new log entries (otherwise a system crash will leave a kernel without a consistent set of log entries in both transaction log files), so the partial reconcile process is initiated. The partial reconcile process writes current unreconciled data to a primary file, but doesn't update its base block (so the sequence numbers are the same as before). Thus, there is no overhead related to updating the base block twice (to validate the sequence numbers and to immediately invalidate them for a new log entry).

Typically, the first log entry in a transaction log file to be applied first has a sequence number equal to a secondary sequence number of the base block in a primary file and this sequence number is incremented by 1 for each subsequent log entry (even when there was a switch to another transaction log file). When the partial reconcile process has finished and the transaction log file has been rotated twice, there will be missing sequence numbers in a transaction log file to be applied first.

## Example
The initial state of registry files:
* Secondary sequence number of the base block in the primary file: 100.
* The first transaction log file (.LOG1) contains log entries with sequence numbers within the following range: 100-200.
* The second transaction log file (.LOG2) contains no active log entries.

After switching to the second transaction log file (.LOG2), the following state can be observed:
* Secondary sequence number of the base block in the primary file: 100.
* The first transaction log file (.LOG1) contains log entries with sequence numbers within the following range: 100-200.
* The second transaction log file (.LOG2) contains log entries with sequence numbers within the following range: 201-250.

After switching to the first transaction log file (.LOG1) with the partial reconcile process, the following state can be observed:
* Secondary sequence number of the base block in the primary file: 100.
* The first transaction log file (.LOG1) contains log entries with sequence numbers within the following range: 251-300.
* The second transaction log file (.LOG2) contains log entries with sequence numbers within the following range: 201-250.

In the last state, log entries with sequence numbers within the range 100-200 are missing. Also, the primary file contains data as of the log entry #250 (although its base block contains a secondary sequence number equal to 100).

# Consequences
As shown above, the first log entry to be applied to a dirty primary file may contain a sequence number which isn't equal to a secondary sequence number of the base block in a primary file.
As a result, since a primary file contains more recent data than log entries to be applied first, these log entries will bring the hive to the "previous state" when applied. And this "previous state" of the hive may be inconsistent, because not all pages contain "previous data".

# Experimental testing
The test was conducted using an Insider Preview build of Windows 10 "Redstone 4" installed in a virtual machine. The operating system installation was running a program calling the *RegFlushKey()* function for the SOFTWARE hive in an infinite loop (to generate many log entries within a short time-frame). On a host machine, scripts to extract and analyze the SOFTWARE, SOFTWARE.LOG1, and SOFTWARE.LOG2 files from the \Windows\System32\config\ directory (using The Sleuth Kit) were launched after the guest operating system has finished booting.

The following effects had been observed:
* The hive writer was initially using the SOFTWARE.LOG1 file to store log entries.
* After some time, the hive writer switched to the SOFTWARE.LOG2 file.
* At the end of the test, the hive writer switched back to the SOFTWARE.LOG1 file.

Before switching back to the SOFTWARE.LOG1 file, the following sequence numbers had been observed (the *before* state):
* Primary file (SOFTWARE), secondary sequence number: 1083.
* First transaction log file (SOFTWARE.LOG1), first log entry: 1083.
* Second transaction log file (SOFTWARE.LOG2), first log entry: 1554.

MD5 hash of the primary file before the switch: 4df67ea3abf65a6379d122691db80ef3.

After the switch, the following sequence numbers had been observed (the *after* state):
* Primary file (SOFTWARE), secondary sequence number: 1083.
* First transaction log file (SOFTWARE.LOG1), first log entry: 1965.
* Second transaction log file (SOFTWARE.LOG2), first log entry: 1554.

MD5 hash of the primary file after the switch: 070c79a5e26deb640620a2533c349433.

During the examination of the registry files extracted, the following effects were observed:
* The *after* state: applying the first log entry results in the inconsistent hive.
```python
>>> from yarp import *
>>> f = open('./after/SOFTWARE', 'rb')
>>> l1 = open('./after/SOFTWARE.LOG1', 'rb')
>>> l2 = open('./after/SOFTWARE.LOG2', 'rb')
>>> h = Registry.RegistryHive(f)
>>> def cb():
...   print('Log entry applied')
...   h.walk_everywhere()
... 
>>> h.log_entry_callback = cb
>>> h.recover_auto(None, l1, l2)
Log entry applied
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 303, in recover_auto
    self.recover_new(log1, log2)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 243, in recover_new
    self.registry_file.apply_new_log_files(file_object_log_or_log1, file_object_log2, self.log_entry_callback)
  File "/usr/local/lib/python3.5/dist-packages/yarp/RegistryFile.py", line 1204, in apply_new_log_files
    self.apply_new_log_file(first, callback)
  File "/usr/local/lib/python3.5/dist-packages/yarp/RegistryFile.py", line 1161, in apply_new_log_file
    callback()
  File "<stdin>", line 3, in cb
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 378, in walk_everywhere
    process_key(self.root_key())
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 370, in process_key
    process_key(subkey)
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 366, in process_key
    for value in key.values():
  File "/usr/local/lib/python3.5/dist-packages/yarp/Registry.py", line 695, in values
    list_buf = self.get_cell(list_offset)
  File "/usr/local/lib/python3.5/dist-packages/yarp/RegistryFile.py", line 984, in get_cell
    raise CellOffsetException('There is no valid cell starting at this offset (relative): {}'.format(cell_relative_offset))
yarp.RegistryFile.CellOffsetException: 'There is no valid cell starting at this offset (relative): 66375664'
```
* Applying all log entries from the SOFTWARE.LOG2 file (taken from the *after* state) to the SOFTWARE file taken from the *before* state results in the same hive bins data as present in the SOFTWARE file taken from the *after* state:
```python
>>> from yarp import *
>>> from io import BytesIO
>>> import hashlib
>>> f = open('./before/SOFTWARE', 'rb')
>>> l1 = BytesIO()
>>> l2 = open('./after/SOFTWARE.LOG2', 'rb')
>>> h = Registry.RegistryHive(f)
>>> h.recover_auto(None, l1, l2)
AutoRecoveryResult(recovered=True, is_new_log=True, file_objects=[<_io.BufferedReader name='./after/SOFTWARE.LOG2'>])
>>> h.registry_file.file.file_object.seek(4096)
4096
>>> b = h.registry_file.file.file_object.read()
>>> hashlib.md5(b).hexdigest()
'804c72c08992e8e4dabd8ce67559d683'
```

```
$ dd if=./after/SOFTWARE bs=4096 skip=1 status=none | md5sum
804c72c08992e8e4dabd8ce67559d683  -
```

* Applying all log entries from both transaction log files results in the consistent hive.

```python
>>> from yarp import *
>>> f = open('./after/SOFTWARE', 'rb')
>>> l1 = open('./after/SOFTWARE.LOG1', 'rb')
>>> l2 = open('./after/SOFTWARE.LOG2', 'rb')
>>> h = Registry.RegistryHive(f)
>>> h.recover_auto(None, l1, l2)
AutoRecoveryResult(recovered=True, is_new_log=True, file_objects=[<_io.BufferedReader name='./after/SOFTWARE.LOG1'>, <_io.BufferedReader name='./after/SOFTWARE.LOG2'>])
>>> h.registry_file.last_sequence_number
1982
>>> h.walk_everywhere()
>>> 
```

These effects match what was expected.

## Registry files
The *before* and *after* states of the registry files are available for an examination on GitHub in this directory.

___
© 2018 Maxim Suhanov

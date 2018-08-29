# 14.1 Write a test program that illustrates your system’s behavior when a process is blocked while trying to write lock a range of a file and additional read-lock requests are made. Is the process requesting a write lock starved by the processes read locking the file?
On linux 3.2, the high number readers locks, do infact strave the writer lock requesting process. Solaris 10,  prevents readers locks from being granted if there is waiting write lock requesting process.

One way to examine a file's lock status is using the lsof command on the lock file itself. Below 3 porcess that share a read lock on the file.
```
vagrant@precise64:/vagrant/git_projects/advC$ lsof lock.lock
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
a.out   6499 vagrant    3u   REG   0,20        0  902 lock.lock
a.out   6500 vagrant    3uR  REG   0,20        0  902 lock.lock
a.out   6501 vagrant    3uR  REG   0,20        0  902 lock.lock
a.out   6502 vagrant    3u   REG   0,20        0  902 lock.lock
a.out   6503 vagrant    3uR  REG   0,20        0  902 lock.lock
```
At an other instant a exclusive write lock is held by one of the process.
```
vagrant@precise64:/vagrant/git_projects/advC$ lsof lock.lock
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
a.out   6499 vagrant    3u   REG   0,20        0  902 lock.lock
a.out   6500 vagrant    3uW  REG   0,20        0  902 lock.lock
a.out   6501 vagrant    3u  REG   0,20        0  902 lock.lock
a.out   6502 vagrant    3u   REG   0,20        0  902 lock.lock
a.out   6503 vagrant    3u  REG   0,20        0  902 lock.lock
```
# 14.2 Take a look at your system’s headers and examine the implementation of select and the four FD_ macros.

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
<TODO>
  
# 14.5 Implement the function sleep_us, which is similar to sleep, but waits for a specified number of microseconds. Use either select or poll. Compare this function to the BSD usleep function.
sleep using select.
```
#include <stdio.h>
#include <sys/select.h>
#include <sys/time.h>
#include <stdlib.h>

void select_sleep(long int usec) {
    struct timeval t;
    t.tv_sec = usec/1000000;
    t.tv_usec = usec%1000000;
    select(0, NULL, NULL, NULL, &t);
}

int
main(void) {
    select_sleep(10000000);
    return 0;
}
```
# 14.6 Can you implement the functions TELL_WAIT, TELL_PARENT, TELL_CHILD, WAIT_PARENT, and WAIT_CHILD from Figure 10.24 using advisory record locking instead of signals? If so, code and test your implementation.
The aneswer is ```NO```, we cannot use locks to emulate the program written using signals.
The below program tries to execute the ```parent critical section``` before the ```child critical section```, however, the required inital conditions, where parent proc holds byte 0 lock and child proc holds byte 1 lock cannot be garenteed. 
i.e The fact that forked child has to complete with the parent to obtain its byte-1 lock, which is a inital condition, marks the failure of the alorithm.

```C
#include <stdio.h>
#include <fcntl.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
static int lockfd;

int
lock_reg(int fd, int cmd, int type, off_t offset, int whence, off_t len) {
    struct flock lock;
    lock.l_type =  type;
    lock.l_start = offset;
    lock.l_whence = whence;
    lock.l_len = len;
    return fcntl(fd, cmd, &lock);
}

void TELL_CHILD() {
    //get parent lock;
    lock_reg(lockfd, F_SETLKW, F_UNLCK, 0,SEEK_CUR,1);
}
void TELL_PARENT() {
    //get parent lock;
    lock_reg(lockfd, F_SETLKW, F_UNLCK, 1,SEEK_CUR,1);
}

void WAIT_PARENT() {
    //get parent lock;
    lock_reg(lockfd, F_SETLKW, F_WRLCK, 0,SEEK_CUR,1);
}

void WAIT_CHILD() {
    //get childlock;
    lock_reg(lockfd, F_SETLKW, F_WRLCK, 1,SEEK_CUR,1);
}

int
main(void) {
    lockfd = open("lock.lock", O_WRONLY| O_TRUNC|O_CREAT|O_SYNC, 0666);
    write(lockfd, "01",2);
    //byte 0 is for the parent
    lock_reg(lockfd, F_SETLKW, F_WRLCK, 0,SEEK_CUR,1);
    if(fork() == 0) {
        //byte 1 is for the child
        lock_reg(lockfd, F_SETLKW, F_WRLCK, 1,SEEK_CUR,1);
        WAIT_PARENT();
        fprintf(stderr, "child critical section\n");
        TELL_PARENT();
    }else {
        /*
          THIS IS WHERE THE PROGRAM FAILS. PARENT SLEEP IS INSERTED HERE TO GIVE THE CHILD A CHANCE TO OBTAIN ITS CHILD LOCK.
          ELSE, THE PARENT CAN GET HOLD BOTH LOCKS AND DEADLOCK THE PROGRAM. DEADLOCKS CAN STILL OCCUR DISPITE THE SLEEP.
        */
        sleep(2);  
        fprintf(stderr, "parent critical section\n");
        TELL_CHILD();
        WAIT_CHILD();
    }
    return 0;
}
```
```
vagrant@precise64:/vagrant/git_projects/advC$ 
parent critical section
child critical section
```
# 14.7 Determine the capacity of a pipe using nonblocking writes. Compare this value with the value of PIPE_BUF from Chapter 2.
We try to ascetain the value of the pipe buffer by transfering a large buffer of bytes in non blocking mode. The max bytes written or read at single time gives is the value of PIPE_BUF
```C
#include <stdio.h>
#include <errno.h>
#include <fcntl.h>
#include <stdlib.h>
int
main(void) {
    int PIPESIZE =1000000; //approx 1MB of bytes
    
    //Allocate write and read buffers.
    char *writeptr = (char *)malloc(PIPESIZE); 
    char *readptr = (char *)malloc(PIPESIZE);
    
    int rc,wc;
    int rz = PIPESIZE;
    int wz = PIPESIZE;
    
    //pipe file desp. p[0] for reading and p[1] for writing.
    int p[2];
    
    //Setup the pipe
    if(pipe(p) < 0 ) {
        perror("");
        exit(-1);
    }
    fcntl(p[0], F_SETFL, O_NONBLOCK);
    fcntl(p[1], F_SETFL, O_NONBLOCK);
    while(1){
        if(wz > 0) {
            wc = write(p[1], writeptr, wz);
            writeptr += wc;
            wz-=wc;
        }
        if(rz > 0) {
            rc = read(p[0], readptr, rz);
            readptr+= rc;
            rz-=rc;
        }
        fprintf(stderr,"readcount %d writecount %d\n", rc, wc);
        if(wz<=0 && rz<=0)
            break;
             }
    free(writeptr - PIPESIZE);
    free(readptr - PIPESIZE);
    return 0;
}
```
```
vagrant@precise64:/vagrant/git_projects/advC$ ./a.out
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 65536 writecount 65536
readcount 16960 writecount 16960
```
The max number size of read or writes in non-blocking mode is ```65536``` which also happens to be the size of ```PIPE_BUF``` on a linux 3.2 system. 

# 14.8 Rewrite the program in Figure 14.21 to make it a filter: read from the standard input and write to the standard output, but use the asynchronous I/O interfaces. What must you change to make it work properly? Keep in mind that you should get the same results whether the standard output is attached to a terminal, a pipe, or a regular file.

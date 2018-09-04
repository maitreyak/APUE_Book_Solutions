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
The problem is that forked child has to compete with the parent to obtain the byte-1 lock, which is a inital condition, marks the failure of the alorithm.

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

File sizes are not relevent when we deal with Pipes or streams. Instead of using fstat and relying on the file size. We use a flag that turned off the momnet we detect 0 bytes on the input stream. This gives the program a generic way of handleing stream, files or pipes.
```C
#include <aio.h>
#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>
#include <errno.h>
#include <unistd.h>

#define BSZ 100  //Reduced the buffer size to demo more aync operations.
#define NBUF 8

enum rwop {
	UNUSED = 0,
	READ_PENDING = 1,
	WRITE_PENDING = 2
};

struct buf {
	enum rwop rwop;
	struct aiocb aiocb;
	unsigned char data[BSZ];
};

struct buf bufs[NBUF];

int
main(int argc, char *argv[]) {
	if(argc< 3) {
		fprintf(stderr, "Not enought args\n");
	}
	int ifd, ofd, i, n, err, off=0, numops = 0 ; 
	/*
		File sizes are not relevent when we deal with Pipes or streams.
		Instead of using fstat and relying on the file size. We use a flag that turned off the momnet we detect 0 bytes
		on the input stream. This gives the program a generic way of handleing stream, files or pipes.
	*/
	int ALLOW_READS = 1;
					
	ifd = open(argv[1], O_RDONLY);
	ofd = open(argv[2], O_RDWR|O_CREAT|O_TRUNC, 0666);
	
	//ifd = STDIN_FILENO;
	//ofd = STDOUT_FILENO;
	
	struct aiocb const *aiolist[NBUF];

	for(i=0; i < NBUF; i++) {
		bufs[i].aiocb.aio_buf = bufs[i].data;
		bufs[i].rwop = UNUSED;
		bufs[i].aiocb.aio_sigevent.sigev_notify = SIGEV_NONE;
		aiolist[i]= NULL;
	}

	for (;;) {
		for (i=0 ; i< NBUF; i++) {
			switch(bufs[i].rwop){
				case UNUSED:
					if(ALLOW_READS) { 
						bufs[i].rwop = READ_PENDING;
						bufs[i].aiocb.aio_fildes = ifd;
						bufs[i].aiocb.aio_offset = off;
						bufs[i].aiocb.aio_nbytes = BSZ;
						off += BSZ;
						aio_read(&bufs[i].aiocb);
						aiolist[i] = &bufs[i].aiocb;
						numops++;
					}
					break;			

				case READ_PENDING:
					if ((err = aio_error(&bufs[i].aiocb)) == EINPROGRESS ) {
						continue;
					} 
					if ((n = aio_return(&bufs[i].aiocb)) < 0) {
						perror("read error");
						exit(-1);
					}
					if(n == 0) {
						bufs[i].rwop = UNUSED;
						ALLOW_READS = 0;
						numops--;
						break;
					}
					/*
						ROT13 transform funtion goes here. Not implemented due to trivial nature of functionlity.
						Doing a direct copy instead.
					*/
					bufs[i].rwop = WRITE_PENDING;
					bufs[i].aiocb.aio_fildes = ofd;
					bufs[i].aiocb.aio_nbytes = n;
					if(aio_write(&bufs[i].aiocb) < 0) {
						perror("write error\n");
						exit(-1);
					}
					break;
				
				case WRITE_PENDING:
					if ((err = aio_error(&bufs[i].aiocb)) == EINPROGRESS ) {
						continue;
					} 
					if ((n = aio_return(&bufs[i].aiocb)) < 0) {
						perror("read error");
						exit(-1);
					}
					bufs[i].rwop = UNUSED;
					aiolist[i] = &bufs[i].aiocb;
					numops--;
					break;
			}
		}			

		if(ALLOW_READS == 0 && numops == 0) {
			//we are done with all the async operations we triggered
			break;
		}else {
			//wait for all the operations on the list to complete
			aio_suspend(aiolist, NBUF, NULL);
		}
	}
	//wait till all the writes complete before exiting.
	bufs[0].aiocb.aio_fildes = ofd;
	aio_fsync(O_SYNC, &bufs[0].aiocb);
	return 0;
}
```
Notice with the new implmentation the bytes copied is equal. Verifying the program.
```
vagrant@precise64:/vagrant/git_projects/advC$ gcc -g async_filter.c -lrt
vagrant@precise64:/vagrant/git_projects/advC$ ./a.out /etc/services temp.temp 2>err.err
vagrant@precise64:/vagrant/git_projects/advC$ ll /etc/services temp.temp
-rw-r--r-- 1 root    root    19281 Feb 13  2012 /etc/services
-rw-r--r-- 1 vagrant vagrant 19281 Sep  4 21:42 temp.temp
```
# 14.9 Recall Figure 14.23. Determine the break-even point on your system where using writev is faster than copying the data yourself and using a single write.

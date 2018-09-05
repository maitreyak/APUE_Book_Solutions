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

# 14.3 The system headers usually have a built-in limit on the maximum number of descriptors that the fd_set data type can handle. Assume that we need to increase this limit to handle up to 2,048 descriptors. How can we do this?
<TODO>

# 14.4 Compare the functions provided for signal sets (Section 10.11) and the fd_set descriptor sets. Also compare the implementation of the two on your system.
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
The can be used to demonstrate that writev can perform better when compared to single writes. 

```C
#include <time.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <stdio.h>
#include <sys/uio.h>
#include <stdlib.h>

#define VECTOR_COUNT 2024
#define VSIZE 4024
void timespec_diff(struct timespec *start, struct timespec *stop,
                   struct timespec *result)
{
    if ((stop->tv_nsec - start->tv_nsec) < 0) {
        result->tv_sec = stop->tv_sec - start->tv_sec - 1;
        result->tv_nsec = stop->tv_nsec - start->tv_nsec + 1000000000;
    } else {
        result->tv_sec = stop->tv_sec - start->tv_sec;
        result->tv_nsec = stop->tv_nsec - start->tv_nsec;
    }
    return;
}


int
main(void) {
	struct iovec vectors[VECTOR_COUNT];
	struct timespec start, end, result;
	int i =0;
	for (i=0 ; i<VECTOR_COUNT; i++) {
		vectors[i].iov_base = malloc(VSIZE);
		vectors[i].iov_len = VSIZE;
		memset(vectors[i].iov_base, "a" ,VSIZE);
	}
	int fdout = open("temp.temp", O_CREAT|O_WRONLY|O_TRUNC);
	int fdout1 = open("temp1.temp", O_CREAT|O_WRONLY|O_TRUNC);

	clock_gettime(CLOCK_REALTIME, &start);
	writev(fdout, vectors, VECTOR_COUNT);
	clock_gettime(CLOCK_REALTIME, &end);
	timespec_diff(&start, &end, &result);
	printf("Real time: Mutiple buffer writes: writev COUNT %d SIZE %d: %lld.%.9ld\n", 
				VECTOR_COUNT, VSIZE,(long long)result.tv_sec, result.tv_nsec);

	void *ptr = malloc(VECTOR_COUNT*VSIZE);
	memset(ptr,"a" ,VECTOR_COUNT*VSIZE);

	clock_gettime(CLOCK_REALTIME, &start);
	write(fdout1,ptr,VECTOR_COUNT*VSIZE);
	clock_gettime(CLOCK_REALTIME, &end);
	timespec_diff(&start, &end, &result);

	printf("Real time: Single buffer of size %d write: %lld.%.9ld\n", 
				VECTOR_COUNT * VSIZE, (long long)result.tv_sec, result.tv_nsec);
	return 0;
}
```
```
vagrant@precise64:/vagrant/git_projects/advC$ ./a.out
Real time: Mutiple buffer writes: writev COUNT 1024 SIZE 4024: 0.173320540
Real time: Single buffer of size 4120576 write: 0.045053265	
```
When the single buffer size is increses, a single write does worse. 
```writev``` does better , when we equivalently increase the number buffers to match the single large buffer.
```
vagrant@precise64:/vagrant/git_projects/advC$ ./a.out
Real time: Mutiple buffer writes: writev COUNT 2024 SIZE 4024: 0.000010054
Real time: Single buffer of size 8144576 write: 0.089522168
```
# 14.10 Run the program in Figure 14.27 to copy a file and determine whether the last-access time for the input file is updated.

No the program does not update the last access time. refer https://github.com/maitreyak/apue-exercises/blob/master/exercises.md#1411


# 14.11 In the program from Figure 14.27, close the input file after calling mmap to verify that closing the descriptor does not invalidate the memory-mapped I/O.
```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h> 
#include <fcntl.h>
#include <sys/mman.h>
#include <errno.h>

#define COPYINCR (1024*1024*1024)       /* 1 GB */
#define FILE_MODE 0666

int
main(int argc, char *argv[])
{
    int             fdin, fdout;
    void            *src, *dst;
    size_t          copysz;
    struct stat     sbuf;
    off_t           fsz = 0;

    if (argc != 3){
        fprintf(stderr,"usage: %s <fromfile> <tofile>", argv[0]);
		exit(-1);
	}
    if ((fdin = open(argv[1], O_RDONLY)) < 0){
        fprintf(stderr,"can't open %s for reading", argv[1]);
		exit(-1);
	}

    if ((fdout = open(argv[2], O_RDWR | O_CREAT | O_TRUNC,
            FILE_MODE)) < 0){
        fprintf(stderr,"can't creat %s for writing", argv[2]);
		exit(-1);
	}

    if (fstat(fdin, &sbuf) < 0){   /* need size of input file */
        fprintf(stderr,"fstat error");
		exit(-1);
	}

    if (ftruncate(fdout, sbuf.st_size) < 0) { /* set output file size */
        fprintf(stderr,"ftruncate error");
		exit(-1);
	}

    while (fsz < sbuf.st_size) {
        if ((sbuf.st_size - fsz) > COPYINCR)
            copysz = COPYINCR;
        else
            copysz = sbuf.st_size - fsz;


        if ((src = mmap(0, copysz, PROT_READ, MAP_SHARED,
            fdin, fsz)) == MAP_FAILED ) {
                fprintf(stderr,"mmap error for input");
				exit(-1);
		}
        
         Closing input file as per the question.
        if (close(fdin) < 0) {
            fprintf(stderr,"couldn't close fdin");
			exit(-1);
		}

        if ((dst = mmap(0, copysz, PROT_READ | PROT_WRITE,
            MAP_SHARED, fdout, fsz)) == MAP_FAILED) {
                fprintf(stderr,"mmap error for output");
			exit(-1);
		}

        memcpy(dst, src, copysz);       /* does the file copy */
        munmap(src, copysz);
        munmap(dst, copysz);
        fsz += copysz;
    }
    exit(0);
}
```
The program works as expected even after the fd is closed. refer https://github.com/maitreyak/apue-exercises/blob/master/exercises.md#1411

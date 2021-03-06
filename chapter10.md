# 10.1 In Figure 10.2, remove the for (;;) statement. What happens and why?
```c
#include "apue.h"

static void	sig_usr(int);	/* one handler for both signals */

int
main(void)
{
	if (signal(SIGUSR1, sig_usr) == SIG_ERR)t
		err_sys("can't catch SIGUSR1");
	if (signal(SIGUSR2, sig_usr) == SIG_ERR)
		err_sys("can't catch SIGUSR2");
		pause();
}

static void
sig_usr(int signo)		/* argument is signal number */
{
	if (signo == SIGUSR1)
		printf("received SIGUSR1\n");
	else if (signo == SIGUSR2)
		printf("received SIGUSR2\n");
	else
		err_dump("received signal %d\n", signo);
}
```
Pause() puts the calling process to sleep and waits for the process to receive a signal. Without the ```for(;;)``` the program only waits for one signal and then terminates.

# 10.2 Implement the sig2str function described in Section 10.22.
Implemention using strsignal.
```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <error.h>

int
sig2str(int signo, char *str) {
    if(signo < 0 || signo > NSIG)
        return -1;
    strcpy(str , strsignal(signo));
    return 0;
}

int
main(void) {
    char buf[100];
    if(sig2str(20, buf) < 0 ){
        exit(-1);
    }
    printf("%s\n",buf);
    exit(0);
}
```
```
vagrant@precise64:/vagrant/git_projects/advC$ ./a.out
Stopped
```
# 10.3 Draw pictures of the stack frames when we run the program from Figure 10.9.
```c
#include <stdio.h>
#include <signal.h>
#include <setjmp.h>
#include <stdlib.h>

static jmp_buf env;

void alarmHandler(int signo) {
    printf("inside alarm handler!\n");
    longjmp(env, 1);
}

void intHandler(int signo) {
    int i,j,k;
    printf("entering int handlers\n");
    for(i=0; i< 300000; i++)
        for(j=0; j< 300000; j++)
            k += i * j;
    printf("exiting int handlers\n");
}

int sleep2(int secs) {
    signal(SIGINT, intHandler);
    signal(SIGALRM, alarmHandler);

    if(setjmp(env) == 0) {
        alarm(secs);
        pause();
    }
    return alarm(0);
}

int
main(void) {
    sleep2(5);
    printf("sleep2 done!\n");
}
```
Using gdb to viszulize the frames. Place a breakpoint at ```longjmp``` and ```setjmp``` line. 
```
(gdb) bt
#0  alarmHandler (signo=14) at sigstacks.c:10
#1  <signal handler called>
#2  0x00000000004006a1 in intHandler (signo=2) at sigstacks.c:16
#3  <signal handler called>
#4  0x00007ffff7adafc0 in pause () from /lib/x86_64-linux-gnu/libc.so.6
#5  0x00000000004006f3 in sleep2 (secs=5) at sigstacks.c:26
#6  0x0000000000400712 in main () at sigstacks.c:35
```

At breakpoint in line ```if(setjmp(env) == 0)```, as we know is the longjmp destination. On running backtrace(bt), we notice the stack has instantly shrunk to just 2 frames.

```(gdb) bt
#0  sleep2 (secs=5) at sigstacks.c:24
#1  0x0000000000400712 in main () at sigstacks.c:35
```
Most noticabley, frames related to intSignalHandler are gone among other frames. Therefore, we never see the intHandler ever finish. 
``` setjmp``` and ```longjmp``` are risky operations that mess with the process stack and hence must be used with great care.
# 10.4 In Figure 10.11, we showed a technique that’s often used to set a timeout on an I/O operation using setjmp and longjmp. The following code has also been seen: What else is wrong with this sequence of code?
```
signal(SIGALRM, sig_alrm);
alarm(60);
if (setjmp(env_alrm) != 0) {     
	/* handle timeout */      
	...
}
```
Assuming sig_alrm signal handler function uses ```longjmp``` using the ```jmp_buf env_alrm```. However, based on how process scheduling plays out their is chnace that SIGALRM is invoked even before the ```jmp_buf env_alrm``` in initilized. In that can the signal handler ```longjmp``` attempting to jump using uninitialized ```jmp_buf env_alrm``` will possibly SEGFAULT or worse in the process.

# 10.5 Using only a single timer (either alarm or the higher-precision setitimer), provide a set of functions that allows a process to set any number of timers.
```C
/*
Since we are intrested in SIGNALS in this excercise, implemetation below uses child process to setup multiple alarms rather than maintaining a data structure to keep track alarams. 
*/
#include <stdio.h>
#include <signal.h>
#include <stdlib.h>
/*
	The OPTIONAL parent SIGALRM handler. Does nothing useful apart from logging the SIGALRM received from the child.
*/
void parent_alrm_handler(int signo) {
	printf("Parent proc caught alrm signal\n");
	return;
}

/*
	The child SIGALRM handler.Replays the SIGALRM back to the parent when it receives it.
*/
void child_alrm_handler(int signo){
	printf("sending kill SIGALRM to the parent %d\n", getppid());
	kill(getppid(), SIGALRM);
}

/*
	Since each proc has single timer. We create child proceses when the req for new alarms is made.
	Caveat: This goes create zombie process. Handling  the it would be overkill for the current example.
*/
void set_alarm(int secs){
	signal(SIGALRM, parent_alrm_handler);
	if( fork() == 0 ){
		signal(SIGALRM, child_alrm_handler);
		alarm(secs);
		printf ("setting an alarm\n");
		pause();
		exit(0);		
	}	
}

int
main(void) {
	set_alarm(1); //alarm 1		
	set_alarm(4); //alarm 2		
	for(;;)
		pause();
	return 0;
}

```
# 10.6 Write the following program to test the parent–child synchronization functions in Figure 10.24. The process creates a file and writes the integer 0 to the file. The process then calls fork, and the parent and child alternate incrementing the counter in the file. Each time the counter is incremented, print which process (parent or child) is doing the increment.

```C
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

static volatile sig_atomic_t sigflag;
static sigset_t newmask, oldmask, zeromask;

static void
sig_usr(int signo) {
	sigflag = 1; 
}

void
TELL_WAIT(void) {
	signal(SIGUSR1, sig_usr);
	signal(SIGUSR2, sig_usr);
	sigemptyset(&zeromask);
	sigemptyset(&newmask);
	sigaddset(&newmask, SIGUSR1);
	sigaddset(&newmask, SIGUSR2);
	sigprocmask(SIG_BLOCK, &newmask, &oldmask);
}

void
TELL_PARENT(pid_t pid) {
	kill(pid, SIGUSR2);
}

void
TELL_CHILD(pid_t pid) {
	kill(pid, SIGUSR1);
}

void
WAIT_PARENT(void) {
	while(sigflag == 0) {
		sigsuspend(&zeromask);
	}
	sigflag = 0;
	sigprocmask(SIG_SETMASK, &oldmask, NULL);
}

void
WAIT_CHILD(void) {
	while(sigflag == 0) {
		sigsuspend(&zeromask);
	}
	sigflag = 0;
	sigprocmask(SIG_SETMASK, &oldmask, NULL);
}

/*
	The critical section of the program.
*/
void critical_section(const char* proc, int fd, char *rbuf, char *wbuf ) {
	int value;
	memset(rbuf,0,10);
	memset(wbuf,0,10);
	lseek(fd,0,SEEK_SET);
	read(fd,rbuf, 10);	
	value = atoi(rbuf) + 1;
	sprintf(wbuf,"%d\0", value);
	printf("%s put %s\n",proc , wbuf);
	lseek(fd,0,SEEK_SET);
	write(fd, wbuf,strlen(wbuf));
	return;
}

/*
	The main driver program.	
*/
int
main(void){
	pid_t pid;
	char rbuf[10];
	char wbuf[10];
	//init the file with int 0
	int fd = open("numberFile", O_CREAT|O_WRONLY|O_TRUNC, 0644);
	//write a plain 0 first.
	lseek(fd,0,SEEK_SET);
	write(fd, "0", 1);
	close(fd);
	TELL_WAIT();

	if( (pid = fork()) == 0){ //Child process.
		fd = open("numberFile", O_RDWR|O_SYNC, 0644);
		while(1) {
			critical_section("CHILD", fd, rbuf, wbuf);
			TELL_PARENT(getppid());
			WAIT_PARENT();
			TELL_WAIT();
		}			
	} else {
		fd = open("numberFile", O_RDWR|O_SYNC, 0644);
		while(1) {
			WAIT_CHILD();
			TELL_WAIT();
			critical_section("PARENT", fd, rbuf, wbuf);
			TELL_CHILD(pid);
		}
	}
	return 0;
}

```
Console output
```
PARENT put 3936
CHILD put 3937
PARENT put 3938
CHILD put 3939
PARENT put 3940
CHILD put 3941
PARENT put 3942
CHILD put 3943
PARENT put 3944
CHILD put 3945
PARENT put 3946
CHILD put 3947
PARENT put 3948
CHILD put 3949
PARENT put 3950
CHILD put 3951
PARENT put 3952
CHILD put 3953
PARENT put 3954
CHILD put 3955
PARENT put 3956
^C
vagrant@precise64:/vagrant/git_projects/advC$ cat numberFile
3956
```
# 10.7 In the function shown in Figure 10.25, if the caller catches SIGABRT and returns from the signal handler, why do we go to the trouble of resetting the disposition to its default and call kill the second time, instead of simply calling _exit?
The expecation of the abort function is to get the process to terminate abnormally. Calling _exit or exit instead will get the process to terminate normally.

# 10.8 Why do you think the siginfo structure (Section 10.14) includes the real user ID, instead of the effective user ID, in the si_uid field?
The effective id is either root or the owner of the signal receiving process, which is poor information of the signal generating process. On the other hand, real user Id of signal generating process is far more useful info.
	
# 10.9 Rewrite the function in Figure 10.14 to handle all the signals from Figure 10.1. The function should consist of a single loop that iterates once for every signal in the current signal mask (not once for every possible signal).
```C
#include <stdio.h>
#include <signal.h>
#include <stdlib.h>
#include <string.h>
/*
    Linux Non portable implemtation of based on /usr/include/x86_64-linux-gnu/bits/sigset.h
    The function examines the first 64 bits of the sigset mask and prints the signals.
*/
void pr_mask(sigset_t sigmask){
    unsigned long int mask =  sigmask.__val[0];
    int count = 1;
    while(mask > 0 ) {
        count++;
        if((mask >>= 1) % 2 == 1 ){
            printf("|%s|",strsignal(count));
        }
    }
    printf("\n");
}


int
main(void) {
    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGABRT);
    sigaddset(&mask, SIGUSR1);
    sigaddset(&mask, SIGUSR2);
    sigaddset(&mask, SIGPOLL);
    pr_mask(mask);
}
```
```
|Aborted||User defined signal 1||User defined signal 2||I/O possible|
```
# 10.10 Write a program that calls sleep(60) in an infinite loop. Every five times through the loop (every 5 minutes), fetch the current time of day and print the tm_sec field. Run the program overnight and explain the results. How would a program such as the cron daemon, which runs every minute on the minute, handle this situation?
Since there is inherit delay in dealing with pending signals, long running processes with call such as ```sleep(60)``` drift. Cron handles by setting the alarm slightly earlier ex: ```sleep(59)``` on the next run, if the previous run incured a time drift.   

# 10.11 Modify Figure 3.5 as follows: (a) change BUFFSIZE to 100; (b) catch the SIGXFSZ signal using the signal_intr function, printing a message when it’s caught, and returning from the signal handler; and (c) print the return value from write if the requested number of bytes wasn’t written. Modify the soft RLIMIT_FSIZE resource limit (Section 7.11) to 1,024 bytes and run your new program, copying a file that is larger than 1,024 bytes. (Try to set the soft resource limit from your shell. If you can’t do this from your shell, call setrlimit directly from the program.) Run this program on the different systems that you have access to. What happens and why?
Under Linux 3.2.0, Mac OS X 10.6.8, and Solaris 10, the signal handler for SIGXFSZ is never called. But write returns a count of 24 as soon as the file’s size reaches 1,024 bytes.If we attempt an additional write at the current file offset (the end of the file), we will receive SIGXFSZ and write will fail, returning –1 with errno set to EFBIG.

# 10.12 Write a program that calls fwrite with a large buffer (about one gigabyte). Before calling fwrite, call alarm to schedule a signal in 1 second. In your signal handler, print that the signal was caught and return. Does the call to fwrite complete? What’s happening?
```C
#include <stdio.h>
#include <signal.h>
#include <stdlib.h>
#include <errno.h>

void sigHandler(int signo) {
    printf("Caught Alarm\n");
}

int main () {
    FILE *fp;
    int BYTES = 1000000000;
    void *ptr = malloc(BYTES);
    fp = fopen( "file.txt" , "w" );
    signal(SIGALRM, sigHandler);
    alarm(1);
    if(fwrite(ptr , 1 , BYTES , fp ) != BYTES) {
        fprintf(stderr,"fwrite was interupted");
        exit(errno);
    }
    printf("fwrite completed successfully");
    free(ptr);
    fclose(fp);
    return(0);
}
```
On linux 3.2, the fwrite completes without getting interupted, emperically. However, no signals are blocked during the process and instead /prod/<pid>/status for the running process show the signal is in pending state while the write occurs.
```
vagrant@precise64:/vagrant/git_projects/advC$ cat /proc/`pgrep a.out`/status
Name:	a.out
State:	R (running)
Tgid:	4556
Pid:	4556

Threads:	1
SigQ:	1/2782
SigPnd:	0000000000000000
ShdPnd:	0000000000002000
SigBlk:	0000000000000000
SigIgn:	0000000000000000
SigCgt:	0000000000002000
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	ffffffffffffffff
```
```
vagrant@precise64:/vagrant/git_projects/advC$ a.out &
Caught Alarm
fwrite completed successfully
[2]-  Done                    ./a.out
```
Therefore, the behaviour of fwrite and signals appears implmentation specfic.

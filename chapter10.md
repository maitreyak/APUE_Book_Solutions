# 10.1 
# In Figure 10.2, remove the for (;;) statement. What happens and why?
```
#include "apue.h"

static void	sig_usr(int);	/* one handler for both signals */

int
main(void)
{
	if (signal(SIGUSR1, sig_usr) == SIG_ERR)
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

# 10.2 
# Implement the sig2str function described in Section 10.22.
Implemention using strsignal.
```
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
# 10.3 
# Draw pictures of the stack frames when we run the program from Figure 10.9.
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

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

#10.2 
#Implement the sig2str function described in Section 10.22.

# 8.1 
# In Figure 8.3, we said that replacing the call to _exit with a call to exit might cause the standard output to be closed and printf to return –1. Modify the program to check whether your implementation behaves this way. If it does not, how can you simulate this behavior?
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

void closeStdOut(void) {
    fclose(stdout);
}

int globval = 6;
int
main (void) {
    int var;
    pid_t pid;

    var = 88;
    printf("before vfork\n");
    if( (pid = vfork()) < 0 ) {
    } else if (pid == 0){
        globval++;
        var++;
        atexit(closeStdOut); //close the stdout stream on exit.
        exit(0);
    }
    printf("pid = %ld, glob = %d var = %d\n", (long)getpid(), globval, var); //Output will not show up.
    exit(0);
}
```
```
vagrant@precise64:/vagrant/advC$ gcc vfork.c
vagrant@precise64:/vagrant/advC$ ./a.out
before vfork
```
The last printf from the parent does not show up as the child has closed the shared stdout stream. 
# 8.2 
# Recall the typical arrangement of memory in Figure 7.6. Because the stack frames corresponding to each function call are usually stored in the stack, and because after a vfork the child runs in the address space of the parent, what happens if the call to vfork is from a function other than main and the child does a return from this function after the vfork? Write a test program to verify this, and draw a picture of what’s happening.
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <error.h>


pid_t callVfork(void) {
    pid_t pid;
    if( (pid = vfork()) < 0 ) {
        perror("vfork error");
        exit(0);
    } else if (pid == 0){
        return 0;
    }
    return pid;
}

int globval = 6;
int
main (void) {
    pid_t pid;
    int var;
    var = 88;
    printf("before vfork\n");
    pid = callVfork();
    if ( pid == 0 ){
        printf("Child after vfork funtion return\n");
    } else {
        printf("Parent after vfork funtion return\n");
    }
    exit(0);
}
```
```
vagrant@precise64:/vagrant/advC$ ./a.out
before vfork
Child after vfork funtion return
Segmentation fault
```
vfork behaves unpredictably when return is used. We end up with segfaults when parent process attempts to execute. Only _exit and exec are valid for the child process to end its execution. 
# 8.3 
# Rewrite the program in Figure 8.6 to use waitid instead of wait. Instead of calling pr_exit, determine the equivalent information from the siginfo structure.
Implmentation using the waitid function. 
```c
#include <stdio.h>
#include <sys/wait.h> 
#include <error.h>
#include <stdlib.h>

void
pr_exit	(int status) {
	if (WIFEXITED(status)) {
		printf("Normal termination %d\n", WEXITSTATUS(status));
	}	
	else if (WIFSIGNALED(status)){
		printf("abnormal termination, signal number %d%s\n", WTERMSIG(status),
			#ifdef WCOREDUMP
			  	WCOREDUMP(status)? "Core dump generated":""
			#else
			  	""	
			#endif
		);
	}
	else if(WIFSTOPPED(status)) {
		printf("Process stopped %d\n", WSTOPSIG(status));
	}	
}

int
main(void) {
	pid_t pid;
	siginfo_t siginfo;

	if( (pid = fork()) < 0){
		perror("fork error");exit(-1);
	} 
	if(pid == 0){
		exit(0); //complete the child <-- child 1
	} 	

	if( (pid = fork()) < 0){
		perror("fork error");exit(-1);
	} 
	if(pid == 0){
		abort(); //abort! <--- child 2
	} 	

	if( (pid = fork()) < 0){
		perror("fork error");exit(-1);
	} 
	if(pid == 0){
		int i = 1/0; //divideByZero <-- child 3
	} 	
	if( (pid = fork()) < 0){
		perror("fork error");exit(-1);
	} 
	if(pid == 0){ // additional child: stop contiue <- child 4
		setbuf(stdout, NULL);
		printf("Infinite loop  CHILD  pid %ld\n", (long)getpid());
		for(;;){
			sleep(1);
		}
	}
	//parent proc to that waits on all its childern.
	while( waitid(P_ALL, -1, &siginfo, WEXITED | WSTOPPED | WCONTINUED) == 0)  { //this is the parent thread
		pr_exit(siginfo.si_status);
	}
	return 0;
}
```
```
vagrant@precise64:/vagrant/advC$ ./parent_proc &
[1] 3619
vagrant@precise64:/vagrant/advC$ abnormal termination, signal number 8
Normal termination 0
Infinite loop  CHILD  pid 3623
abnormal termination, signal number 6

vagrant@precise64:/vagrant/advC$ jobs
[1]+  Running                 ./parent_proc &
vagrant@precise64:/vagrant/advC$ kill -STOP 3623   #SEND STOP SINGNAL
vagrant@precise64:/vagrant/advC$ abnormal termination, signal number 19

vagrant@precise64:/vagrant/advC$ kill -CONT 3623   #SEND CONTINUE SIGNAL
vagrant@precise64:/vagrant/advC$ abnormal termination, signal number 18

vagrant@precise64:/vagrant/advC$ jobs
[1]+  Running                 ./parent_proc & #Parent proc keeps waiting on child until child its terminated.
vagrant@precise64:/vagrant/advC$ kill -9 3623 
vagrant@precise64:/vagrant/advC$ abnormal termination, signal number 9

[1]+  Done                    ./parent_proc
```
# 8.4 
# When we execute the program in Figure 8.13 one time, as in ```$ ./a.out``` the output is correct. But if we execute the program multiple times, one right after the other, as in ```$ ./a.out ; ./a.out ; ./a.out```
```
output from parent
ooutput from parent
ouotuptut from child
put from parent
output from child
utput from child
````

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
# When we execute the program in Figure 8.13 one time, as in 
```$ ./a.out``` 
# the output is correct. But if we execute the program multiple times, one right after the other, as in 
```$ ./a.out ; ./a.out ; ./a.out```
```
output from parent
ooutput from parent
ouotuptut from child
put from parent
output from child
utput from child
````
Notice, the ```$ ./a.out ; ./a.out ; ./a.out``` executes the same program three time in serial order. i.e PARENT_PROC_PID_1 then PARENT_PROC_PID_2 and then PARENT_PROC_PID_3. 

The problem is that PARENT_PROC_<ID> forks a child, executes the print notifies its child and terminates. There the child procs and subsequent PARENT_PROCs executes intermetently. Leading to outputs as above.

However, if we switch the order or execution i.e, PARENT_PROC_<ID> waits for its children to execute and then terminates. In this case, no proc (child or parent) executes in the intermitent fashion garenteeing expected output.

# 8.5 
# In the program shown in Figure 8.20, we call execl, specifying the pathname of the interpreter file. If we called execlp instead, specifying a filename of testinterp, and if the directory /home/sar/bin was a path prefix, what would be printed as argv[2] when the program is run?

```execlp``` looks for the file name in the path prefixes specfied in ```$PATH``` env variable.
Lets execute the program from 
```
vagrant@precise64:~$ pwd
/home/vagrant
```
Lets place the ```testinterp``` interpreter file in ```/vagrant/advC``` directory and add the directory to the ```$PATH``
```
vagrant@precise64:~$ echo $PATH
/usr/lib/jvm/bin::/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/opt/vagrant_ruby/bin:/vagrant/advC
```
Now lets execute the below program 
```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int
main(void) {
    pid_t pid;
    if ((pid == fork()) == 0 ){
        execlp("testinterp", "testinterp", "myarg1", "myarg2" ,(char *)0);
    }
    return;
}
```
execlp now looks for testinterp in the path prefixes in the ```$PATH``` and finds it in ```/vagrant/advC/testinterp``` which is arg[2] as expected. 
```
vagrant@precise64:~$ ./a.out
arg[0]: /vagrant/advC/echoargs
arg[1]: foo
arg[2]: /vagrant/advC/testinterp
arg[3]: myarg1
arg[4]: myarg2
```
# 8.6 
# Write a program that creates a zombie, and then call system to execute the ps(1) command to verify that the process is a zombie.
Using vfork as vfork always lets the child finish. Now the parent does not wait of the child making it [defucnt] a zombie!
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int
main(void) {
    char ptr[10];
    pid_t pid;
    if((pid=vfork()) == 0) {
        _exit(0);//zombie child
    }
    sprintf( ptr, "ps -p %ld",(long) pid);
    system(ptr);
    return 0;
}
```
```
vagrant@precise64:/vagrant/advC$ ./a.out
  PID TTY          TIME CMD
 3843 pts/1    00:00:00 a.out <defunct>
```
# 8.7 
# We mentioned in Section 8.10 that POSIX.1 requires open directory streams to be closed across an exec. Verify this as follows: call opendir for the root directory, peek at your system’s implementation of the DIR structure, and print the close-on-exec flag. Then open the same directory for reading, and print the close-on-exec flag.
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <fcntl.h>
#include <errno.h>
#include <fcntl.h>
#include <dirent.h>

void isCloseOnExec(int fd) {
        int flags;
        if ( (flags = fcntl(fd, F_GETFL, 0)) < 0 ) {
                perror("fcntl error");
                exit(-1);
        }
        if( flags & O_CLOEXEC ){
                printf("close-on-exec is set on DIR\n");
        }else {
                printf("close-on-exec is NOT set on DIR\n");
        }
}

int
main(void) {
        DIR *dir; int fd; int flags;
        pid_t pid;
        dir = opendir("/");
        if(errno != 0) {
                perror("open dir problems");
                exit(-1);
        }

        fd = dirfd(dir);
        isCloseOnExec(fd);

        if((pid = fork()) == 0){
            printf("Child1 proc pid %d\n", getpid());
            execlp("sleep", "sleep", "1000", NULL);
        }

        fd = open("/", O_RDONLY);
        isCloseOnExec(fd);
        if((pid = fork()) == 0){
            printf("Child2 proc pid %d\n", getpid());
            execlp("sleep", "sleep", "1000", (char *)0);
        }
        sleep(1000);
        exit(0);
}
```
```
vagrant@precise64:/vagrant/advC$ ./a.out
close-on-exec is set on DIR
close-on-exec is NOT set on DIR
Child2 proc pid 2864
Child1 proc pid 2863
^Z
[3]+  Stopped                 ./a.out
```
On inspection of the child1 with close on exec enabled. The root ```\``` dir does not appear. 
```
vagrant@precise64:/vagrant/advC$ ll /proc/2863/fd
total 0
dr-x------ 2 vagrant vagrant  0 Aug  2 02:41 ./
dr-xr-xr-x 8 vagrant vagrant  0 Aug  2 02:41 ../
lrwx------ 1 vagrant vagrant 64 Aug  2 02:41 0 -> /dev/pts/0
lrwx------ 1 vagrant vagrant 64 Aug  2 02:41 1 -> /dev/pts/0
lrwx------ 1 vagrant vagrant 64 Aug  2 02:41 2 -> /dev/pts/0
```
On the otherhand, child2 close on exec disabled. We notice that the ```/``` dir is an open desciptor. 
```
vagrant@precise64:/vagrant/advC$ ll /proc/2864/fd
total 0
dr-x------ 2 vagrant vagrant  0 Aug  2 02:41 ./
dr-xr-xr-x 8 vagrant vagrant  0 Aug  2 02:41 ../
lrwx------ 1 vagrant vagrant 64 Aug  2 02:41 0 -> /dev/pts/0
lrwx------ 1 vagrant vagrant 64 Aug  2 02:41 1 -> /dev/pts/0
lrwx------ 1 vagrant vagrant 64 Aug  2 02:41 2 -> /dev/pts/0
lr-x------ 1 vagrant vagrant 64 Aug  2 02:41 4 -> //
```

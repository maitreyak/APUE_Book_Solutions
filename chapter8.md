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
vfork behaves unpredictably when return is used. Only _exit and exec are valid  

# 9.2 
# Write a small program that calls fork and has the child create a new session. Verify that the child becomes a process group leader and that the child no longer has a controlling terminal.
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>

void
controlling_terminal_access() {
    if (open("/dev/tty", O_RDWR) < 0) {
        printf("Child pid is %d wth sid %d has NO LONGER access to the conttolling terminal\n"
                ,getpid(),getsid(getpid()));
    }else {
        printf("Child pid is %d with sid %d has access to the conttolling terminal\n"
                ,getpid(), getsid(getpid()));
    }
}

int
main(void) {
    pid_t pid;
    if ((pid = fork()) == 0){ //child
        controlling_terminal_access();
        setsid(); //new session for the child.
        controlling_terminal_access();
    }
    exit(0);
}
```
```
vagrant@precise64:/vagrant/advC$./a.out 
Child pid is 4702 with sid 4335 has access to the conttolling terminal
Child pid is 4702 wth sid 4702 has NO LONGER access to the conttolling terminal
```

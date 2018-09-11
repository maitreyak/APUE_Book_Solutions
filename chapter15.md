# 15.1 In the program shown in Figure 15.6, remove the close right before the waitpid at the end of the parent code. Explain what happens.
Below is the working but condenced verison of the program that in Fig 15.6
```c 
#include <stdio.h>
#include <fcntl.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/stat.h>

#define DEF_PAGER "/bin/more"
#define MAXLINE 4096

int
main(int argc, char *argv[]) {
    int fd[2];
    int f1  = open(argv[1], O_RDONLY);
    char buf[MAXLINE];
    pid_t pid;
    int count;
    if (pipe(fd) > 0) {
        perror("Pipe error");
        exit(-1);
    }
    if((pid = fork()) > 0) {
        //parent proc
        close(fd[0]);
        while((count  = read(f1, buf, MAXLINE)) > 0) {
            write(fd[1], buf, count);
        }
        //close(fd[1]); commenting out the close. Will fail to genereta the EOF char
        waitpid(pid, NULL, 0);
        exit(0);
    }
    //child proc
    close(fd[1]);
    dup2(fd[0], STDIN_FILENO);
    close(fd[0]);
    execl(DEF_PAGER, "more", (char *) 0);
    return 0;
}
```
Without the close(fd[1]) on the parent proc, the stream does not receive a EOF char, thus the reading child pagination program ```more``` waiting forever after the reading all the contents of the pipe.

# 15.2 In the program in Figure 15.6, remove the waitpid at the end of the parent code. Explain what happens.
Refer the same program as above.
Here when we comment the waitpid, i.e, we do not wait on the child program, it closes both read and write ends of the pipe. Noe considering the child following parents write, there is good chage that of the writes from the parennt are lost to the ```more``` or pagination program.

# 15.3 What happens if the argument to popen is a nonexistent command? Write a small program to test this.
Open forks a shell and the runs ```sh -c cmd```. If the command is not found the shell reports command not found. 
```c
#include <stdio.h>
#define MAXLINE 10000
int
main(void) {
    char buf[MAXLINE];
    FILE *file = popen("nocmd", "r");
    while (fgets(buf, MAXLINE, file) > 0)
        printf("%s\n", buf);
    return 0;
}
```
```
vagrant@precise64:/vagrant/git_projects/advC$ ./a.out
sh: 1: nocmd: not found
```
# 15.4 In the program shown in Figure 15.18, remove the signal handler, execute the program, and then terminate the child. After entering a line of input, how can you tell that the parent was terminated by SIGPIPE?

# 15.5 In the program in Figure 15.18, use the standard I/O library for reading and writing the pipes instead of read and write.


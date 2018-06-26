# 4.1
# Modify the program in Figure 4.3 to use stat instead of lstat. What changes if one of the command-line arguments is a symbolic link?

Replaced stat with lstat and using readlink to detect symbolic link. See comments in the program below.

```c
#include <stdio.h>
#include <errno.h>
#include <sys/stat.h>
#include <stdlib.h>

int
main(int argc, char *argv[])
{

    int         i;
    struct stat buf;
    char        *ptr;
    char        symlink;

    for (i = 1; i < argc; i++) {
        printf("%s: ", argv[i]);
        //using readlink to find the symlink
        if((int)(readlink(argv[i], &symlink, 1)) > 0){
            ptr ="symbolic_link";
            printf("%s\n", ptr);
            continue;
        }
        //replaced lstat with stat
        if(stat(argv[i], &buf) < 0) {
            perror("lstat error");
            continue;
         }

         if (S_ISREG(buf.st_mode))
            ptr = "regular";
         else if (S_ISDIR(buf.st_mode))
            ptr = "directory";
         else if (S_ISCHR(buf.st_mode))
            ptr = "character special";
         else if (S_ISBLK(buf.st_mode))
            ptr = "block special";
         else if (S_ISFIFO(buf.st_mode))
            ptr = "fifo";
         else if (S_ISSOCK(buf.st_mode))
            ptr = "socket";
         else
            ptr = "** unknown mode **";
         printf("%s\n", ptr);
  }
   exit(0);
```
```
vagrant@precise64:/vagrant/advC$ ./a.out file_link file ..
file_link: symbolic_link
file: regular
..: directory
```
# 4.2
# What happens if the file mode creation mask is set to 777 (octal)? Verify the results using your shell’s umask command.
We can demo the umask on the shell and show the details using the **strace**. (my new favourite command)
```
dumdum@precise64:~$ umask 777
dumdum@precise64:~$ strace -eopen touch file1 2>&1 | tail -1
open("file1", O_WRONLY|O_CREAT|O_NOCTTY|O_NONBLOCK, 0666) = 3

dumdum@precise64:~$ ls -l file1
---------- 1 dumdum dumdum 0 Jun 26 03:30 file1
```
**Note:** strace is super handy util that we can use to see all the system calls invoked by a program.

Below is the C program that demos the umask system call.  
```c
#include <stdlib.h>
#include <stdio.h>
#include <fcntl.h>

#define RWXRWXRWX (S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH|S_IXUSR|S_IXGRP|S_IXOTH)
int
main(void){
        umask(0);
        if (creat("foo", RWXRWXRWX) < 0){
                perror("failed creation\n");
                exit(-1);
        }
        umask(RWXRWXRWX);// Is the same as octet 777
        if (creat("bar", RWXRWXRWX) < 0){
                perror("failed creation\n");
                exit(-1);
        }
        return 0;
}
```
```
dumdum@precise64:~$ ./a.out
dumdum@precise64:~$ ls -l foo bar
---------- 1 dumdum dumdum 0 Jun 26 03:25 bar
-rwxrwxrwx 1 dumdum dumdum 0 Jun 26 03:25 foo
```

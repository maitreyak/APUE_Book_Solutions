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

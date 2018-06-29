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
# 4.3
# Verify that turning off user-read permission for a file that you own denies your access to the file.
```
dumdum@precise64:~$ cat file1
cat: file1: Permission denied
dumdum@precise64:~$ ls -l file1
---------- 1 dumdum dumdum 0 Jun 26 03:30 file1
```
# 4.4
# Run the program in Figure 4.9 after creating the files foo and bar. What happens?
The permissions of the foo and bar did not change. Even after the running the program.
```
dumdum@precise64:~$ umask //default umask.
0002
dumdum@precise64:~$ touch foo bar //touch creates premissions 666 & with 002
dumdum@precise64:~$ ls -l foo bar //same as above.
-rw-rw-r-- 1 dumdum dumdum 0 Jun 26 04:31 bar
-rw-rw-r-- 1 dumdum dumdum 0 Jun 26 04:31 foo
/a.out
dumdum@precise64:~$ ls -l foo bar
-rw-rw-r-- 1 dumdum dumdum 0 Jun 26 04:35 bar //No change in permissions.  
-rw-rw-r-- 1 dumdum dumdum 0 Jun 26 04:35 foo //No change in permissions.
```
We observe no change in file permissions. i.e the kernel only applies the umask only during file creation, which is not the case here.

# 4.5
# In Section 4.12, we said that a file size of 0 is valid for a regular file. We also said that the st_size field is defined for directories and symbolic links. Should we ever see a file size of 0 for a directory or a symbolic link?
File sizes of Symbolic links is the length of absolute path that it points to. Therefore, cannot be 0.
Directories point to ```.``` ```..``` in them by default, hence cannot be size 0. 

# 4.6
# Write a utility like cp(1) that copies a file containing holes, without writing the bytes of 0 to the output file.
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>

int
main(int argc, char *argv[]){
    if(argc <3){
        printf("Not enough args");
        exit(-1);
    }
    int BUFFER_SIZE =1024;
    char buffer[BUFFER_SIZE];
    char subbuffer[BUFFER_SIZE];
    int readFd = open(argv[1], O_RDONLY);
    int writeFd = creat(argv[2], O_TRUNC);
    int bytes;
    int i=0,j=0;
    int subBufBytes;
    int holeOffset = 0;

    while((bytes = read(readFd, buffer, BUFFER_SIZE)) > 0 ) {
        subBufBytes = 0; //init subbuffer valid bytes reads;
        for(i=0; i < bytes; i++){
            if( buffer[i] == '\0' ) {
                ++holeOffset;
                continue;
            }

            for(j=i; j< bytes; j++){
                if(buffer[j] == '\0'){
                    break;
                }
                subbuffer[subBufBytes++] = buffer[j];
            }
            i = j-1;
            if(subBufBytes > 0 ) {
                lseek(writeFd, (int)lseek(writeFd,holeOffset, SEEK_CUR), SEEK_SET);
                write(writeFd, subbuffer, subBufBytes);
                holeOffset =0;
            }
        }
    }
return 0;
```
The above copy program reads bytes into src a buffer and then analyzes the buffer to find valid bytes that are written into new file.

```
vagrant@precise64:/vagrant/advC$ cp --sparse=never file.hole file2.hole #for comparison cp file.hole with sparse set to false. 
vagrant@precise64:/vagrant/advC$ ./a.out file.hole file3.hole # Now lets run our program with sparse file detection.
vagrant@precise64:/vagrant/advC$ ll file.hole file2.hole file3.hole # They all report the same size.
-rw-r--r-- 1 vagrant vagrant 100048586 Jun 29 09:32 file2.hole
-rw------- 1 vagrant vagrant 100048586 Jun 29 09:33 file3.hole
-rw-r--r-- 1 vagrant vagrant 100048586 Jun 29 09:26 file.hole
vagrant@precise64:/vagrant/advC$ du file.hole file2.hole file3.hole # du shows the real picture.
8	file.hole
97704	file2.hole
8	file3.hole
```
The alternate approach to the problem would be to use SEEK_HOLE and SEEK_DATA introduced in linux 3.1. The system support would greatly simply the above program.

**Note**
The program to create the sparse file listed in **Example 3.2** will not create a real sparse file on a modern Linux.
That is because the file has to be sufficently large for the kernel/fs to detect a sparse file write.
Therefore change in **Example 3.2** 
```if (lseek(fd, 16384, SEEK_SET) == -1)``` to ```if( lseek(fd, 100048576, SEEK_SET) == -1 )```
to create a sparse file (like file.hole in the above solution).

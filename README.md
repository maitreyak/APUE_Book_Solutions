# APUE_Book_Solutions
Solutions to exercises Advanced Programming in the unix environment 

# 3.1
# When reading or writing a disk file, are the functions described in this chapter really unbuffered? Explain.
The unbuffered read and write systems functions (not system calls mind you) do not use buffer in the user mode. However, their system call counterparts in the kernal mode do. Buffering could be in the form of page caching of files in memory, that are periodically flushed to disk. Therefore, the "unbuffered" functions described do use kernel level buffers. 

# 3.2
# Write your own dup2 function that performs the same service as the dup2 function described in Section 3.12, without calling the fcntl function. Be sure to handle errors correctly.
```c
#include <stdio.h>
#include <fcntl.h>
#include <stdlib.h>
#include <errno.h>

int mydup(const int fd1, const int fd2) {
	int start;
	//check if fd1 exists.
	if((start = dup(fd1)) && errno){
		perror("error in the filedes");
		return -1; 
	}
	//trival case: fd1 = fd2 do nothing return fd1
	if(fd1 == fd2){
		printf("dup complete:new fd is %d\n", fd1);
		return fd1;
	}
	//corner case: fd2 less than the available fd
	if(start > fd2) {
		printf("dup cannot emulate dup2: %d\n", start);
		return start;
	} else if (start == fd2){
		printf("dup success: %d\n", fd2);
		return start;
	}else{
			printf("start from %d dup trying for fd: %d\n", start, fd2);
			int i; int temp;
			//open all the fd from start to dp2
			for(i=start+1;i<=fd2;i++){
				errno = 0;
				temp = dup(fd1);
				printf("duping fd1 %d opened fd %d\n", fd1, temp);
				if(temp == fd2){
					break;
				}
			}	
			//close all the unwanted fd.
			for(i=start ; i<fd2; i++){
				if(i == fd1){
					continue;
				}
				errno = 0;
				printf("closing %d and errno %d\n", i, errno);
				close(i);
			}
			printf("dup success: %d\n", fd2);
			return fd2;	
	}
	return -1;
}


int
main(int argc, char *argv[]){
	if( argc < 2 ){
		printf("Need two file descriptors\n");
		exit(-1);
	}
	int fd;
	if((fd = mydup(atoi(argv[1]), atoi(argv[2]))) < 0){
		printf("failed\n!");
		exit(-1);
	}
	
	write(fd, "SUCCESS\n", 7);
	return 0;
}
```
compile run as below
```
./a.out 5 10 5<>output /* <> tells the shell to open (in read write mode) file on file descriptor 5 for the current pid*/
vagrant@precise64:/vagrant/advC$ ./a.out 5 10 5<>output_file
vagrant@precise64:/vagrant/advC$ cat output_file
SUCCESS
```
During the execution of the program you can check the all open file descriptors by the pid by looking in /proc/{pid}/fd directory. Or just use the lsod -p <pid> command to see all open files (& file descrioptors).

# 3.5
# The Bourne shell, Bourne-again shell, and Korn shell notation ```digit1>&digit2``` says to redirect descriptor digit1 to the same file as descriptor digit2. What is the difference between the two commands ```./a.out > outfile 2>&1 AND ./a.out 2>&1 > outfile``` ?

The differnce between calls can be demonstrated using the following c program and then examining the file descriptor using
the OS.
```c
#include <stdio.h>
#include <stdlib.h>

int
main(void){
    fprintf(stdout, "standard output\n");
    fprintf(stderr, "standard error\n");
    fflush(stdout); /*stdout does not flush immediately, stderr does.*/
    sleep(100); /*sleep gives us ample time to example the filedes on the OS*/
    return 0;
}
```
Run the program and lookup the /proc/{pid}/fd 
```
vagrant@precise64:/vagrant/advC$ ./a.out > output 2>&1
^Z
[1]+  Stopped                 ./a.out > output 2>&1
vagrant@precise64:/vagrant/advC$ ll /proc/2637/fd
total 0
dr-x------ 2 vagrant vagrant  0 Jun 20 20:55 ./
dr-xr-xr-x 8 vagrant vagrant  0 Jun 20 20:55 ../
lrwx------ 1 vagrant vagrant 64 Jun 20 20:55 0 -> /dev/pts/0
l-wx------ 1 vagrant vagrant 64 Jun 20 20:55 1 -> /vagrant/advC/output
l-wx------ 1 vagrant vagrant 64 Jun 20 20:55 2 -> /vagrant/advC/output
```
Therefore, both the standard out and standard error are redirected into the output file.

Next, run the other variation of the program.
```
vagrant@precise64:/vagrant/advC$ ./a.out 2>&1 > output2
standard error
^Z
[2]+  Stopped                 ./a.out 2>&1 > output2
vagrant@precise64:/vagrant/advC$ ll /proc/2658/fd/
total 0
dr-x------ 2 vagrant vagrant  0 Jun 20 20:57 ./
dr-xr-xr-x 8 vagrant vagrant  0 Jun 20 20:57 ../
lrwx------ 1 vagrant vagrant 64 Jun 20 20:57 0 -> /dev/pts/0
l-wx------ 1 vagrant vagrant 64 Jun 20 20:57 1 -> /vagrant/advC/output2
lrwx------ 1 vagrant vagrant 64 Jun 20 20:57 2 -> /dev/pts/0
```
As you can see, "standard error" is outoutted on the console as its file descriptor is still pointing to /dev/pts/0. Standard out has been redirected into file output2.

# 3.6 
# If you open a file for read–write with the append flag, can you still read from anywhere in the file using lseek? Can you use lseek to replace existing data in the file? Write a program to verify this.
Program demonstrates the solution. See answers in the comments. 
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <error.h>

int
main(int argc, char *argv[]) {
    if(argc <2) {
        printf("Need valid file arg");
        exit(-1);
    }
    int fd;
    if((fd = open(argv[1], O_RDWR|O_APPEND)) == -1 ){
        perror("opening file erro");
        exit(-1);
    }
    int current_pos;
    char buf[9];
    if((current_pos = lseek(fd,0,SEEK_SET)) == -1){
        perror("Error seeking");
    }
    read(fd, buf, 8);
    //lseek is set to a cursor location (0 in this case) can read from any line.
    printf("current postion is %d but the sting read is %s\n", current_pos, buf);
    //writes however are only done at the EOF due to the O_APPEND flags.
    write(fd,"endoffile\n",10);
    return 0;
}
```

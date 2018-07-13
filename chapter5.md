# 5.1 
# Implement setbuf using setvbuf.
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static int size_of_buffer(FILE *);
static int is_unbuffered(FILE *);
static int is_line_buffered(FILE *);
static void mysetbuf(FILE *, char *);
static void fp_info(FILE *);

int
main(int argc, char *argv[]) {
    FILE *fp;
    if( (fp = fopen(argv[1], "r")) == NULL) {
        perror("open error");
        exit(-1);
    }
    mysetbuf(fp, (char*) malloc(BUFSIZ));
    fp_info(fp);
    setbuf(fp, NULL);
    fp_info(fp);
    return 0;
}

static void
fp_info(FILE *fp) {
    printf("%d size of buffer\n", size_of_buffer(fp));
    printf("%s is UnBuffered\n", is_unbuffered(fp) > 0 ? "TRUE":"FALSE" );
    printf("%s is LineBuffered\n", is_line_buffered(fp) > 0 ? "TRUE":"FALSE");
    printf("%s is FullyBuffered\n", is_line_buffered(fp) | is_unbuffered(fp) == 0 ? "TRUE":"FALSE");
}

static void 
mysetbuf(FILE *fp, char *buf) {
    setvbuf(fp, buf, buf == NULL ? _IONBF : _IOFBF, BUFSIZ);
    return;
}

static int
size_of_buffer(FILE *fp) {
    return fp->_IO_buf_end - fp->_IO_buf_base;
}

static int
is_unbuffered(FILE *fp) {
    return fp->_flags & _IO_UNBUFFERED;
}

static int is_line_buffered(FILE *fp) {
    return fp->_flags & _IO_LINE_BUF;
}
```
```
vagrant@precise64:/vagrant/advC$ ./a.out file1
8192 size of buffer
FALSE is UnBuffered
FALSE is LineBuffered
TRUE is FullyBuffered

1 size of buffer
TRUE is UnBuffered
FALSE is LineBuffered
FALSE is FullyBuffered
```
# 5.2 
# Type in the program that copies a file using line-at-a-time I/O (fgets and fputs) from Figure 5.5, but use a MAXLINE of 4. What happens if you copy lines that exceed this length? Explain what is happening.
```c
#include <stdio.h>
#include <stdlib.h>
int
main(void)
{
    int maxline = 4;
    char  buf[maxline];
    int count =0;
    while (fgets(buf, maxline, stdin) != NULL){
        if (fputs(buf, stdout) == EOF){
            perror("output error");
            exit(-1);
        }
    }
    if (ferror(stdin)){
        perror("input error");
        exit(-1);
    }
    exit(0);
}
```
Even if the input is larger than the buffer, the loop has do do more work bylooking until it see's EOF(\0) char.
The performce would be slower as the program invokes more write system calls as seen below.
```
read(0, 12345678
"12345678\n", 1024)             = 9
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 0), ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fd22330f000
write(1, "123", 3123)                      = 3
exit_group(0)
```
# 5.3 
# What does a return value of 0 from printf mean?
Printf returns the number of chars before it sees the first ```\0``` passed to the underlying ``write``` system call 
```c
#include <stdio.h>
#include <fcntl.h>
int
main(void) {
    int count = printf("\0therisalotmoredata");
    printf("%d\n",count);
    return 0;
}
```
```
vagrant@precise64:/vagrant/advC$ ./a.out
0
```
Zero is the number of char actually written to the output stream. (stdout)

# 5.4 
# The following code works correctly on some machines, but not on others. What could be the problem?
```c
#include    <stdio.h>

int
main(void)
{
    char    c;

    while ((c = getchar()) != EOF)
        putchar(c);
}
```
System define the default char type is signed char or a unsigned char. If the the default is signed char the program would work normally as EOF converts to ```-1```, which is set properly in an signed type. However, if unsigned is used the the EOF condition never computes to true, thus the program infinite loops, incorrectly, ofcourse.   

# 5.5 
# How would you use the fsync function (Section 3.13) with a standard I/O stream?
fsync needs file descriptor, we can get the fd from the FILE pointer using the ```fileno``` library function.
```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
int main(void){
    FILE *fp;
    if( (fp = fopen("file1", "w+")) < 0 ){
        perror("Open file");
        exit(-1);
    }
    fprintf(fp, "hello there");
    int fd = fileno(fp); //gets the file desc from the file pointer
    fsync(fd); //fsync can now be called easily
    return 0;
}
```
# 5.6 
# In the programs in Figures 1.7 and 1.10, the prompt that is printed does not contain a newline, and we don’t call fflush. What causes the prompt to be output?
fgets flushes the output streams before executing.

# 5.7 
# BSD-based systems provide a function called funopen that allows us to intercept read, write, seek, and close calls on a stream. Use this function to implement fmemopen for FreeBSD and Mac OS X.

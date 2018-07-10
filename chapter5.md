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

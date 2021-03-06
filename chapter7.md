# 7.1 
# On an Intel x86 system under Linux, if we execute the program that prints “hello, world” and do not call exit or return, the termination status of the program — which we can examine with the shell—is 13. Why?
Printf("hello, world\n") returns 13, which is the last return value of the thread which is stored in the ```rax``` register. 
Since  the register is never overwritten, the program reports the same ```rax ``` value as the final return value.

We can examine the value using gdb

Compile the program with the ```-g``` flag.
```
vagrant@precise64:/vagrant/advC$ gcc -g main.c
vagrant@precise64:/vagrant/advC$ gdb a.out
```
Using the breakpoints, view the register ```rax```.
```
(gdb) break main.c:4
Breakpoint 1 at 0x400502: file main.c, line 4.
(gdb) run
Starting program: /vagrant/advC/a.out
Hello, world

Breakpoint 1, main () at main.c:4
4	}
(gdb) info registers rax
rax            0xd	13
```
rax register shows 13.

# 7.2 
# When is the output from the printfs in Figure 7.3 actually output?
After exit() is called and before the _exit() called. i.e exit_handlers.  

# 7.3
# Is there any way for a function that is called by main to examine the command-line arguments without (a) passing argc and argv as arguments from main to the function or (b) having main copy argc and argv into global variables?
A portable soloution is not available. However, on MS-DOS(and possibly windows) environments ,we could access global variables to achive the above. 
```c
extern char	**__argv; 		/* Current argument address	*/
extern int	__argc; 		/* Current argument count	*/
```
Found something similar [online](http://unixjunkie.blogspot.com/2006/07/access-argc-and-argv-from-anywhere.html) for the macOS.
```c
#include <stdio.h>

extern int *_NSGetArgc(void);
extern char ***_NSGetArgv(void);

void DoStuff(void) {
  printf("%s =  %d\n", "_NSGetArgc()", *_NSGetArgc());

  char **argv = *_NSGetArgv();
  for (int i = 0; argv[i] != NULL; ++i)
    printf("%s [%d] = '%s'\n", "_NSGetArgv()", i, argv[i]);
}

int main(void) {
  DoStuff();
  return 0;
}
```
```
$ gcc -std=c99 mac_args.c                                                                                                   $ ./a.out one two three 
_NSGetArgc() =  4
_NSGetArgv() [0] = './a.out'
_NSGetArgv() [1] = 'one'
_NSGetArgv() [2] = 'two'
_NSGetArgv() [3] = 'three' 
```
# 7.4 
# Some UNIX system implementations purposely arrange that, when a program is executed, location 0 in the data segment is not accessible. Why?

```c
#include <stdio.h>
int
main(void) {
    int val =1;
    int *p = NULL;
    printf("0x%08X ptr value\n", p);
    fflush(stdout);
    *(p) = val; // should seg fault here
}
```
```
vagrant@precise64:/vagrant/advC$ ./a.out
0x00000000 ptr value
Segmentation fault
```
Dereferencing NULL which points to data segment location is 0 is considered an runtime error. The firing of the error is enfoced by blocking access to location 0.   

# 7.5 
# Use the typedef facility of C to define a new data type Exitfunc for an exit handler. Redo the prototype for atexit using this data type.
```c
#include <stdio.h>
#include <stdlib.h>

typedef void Exitfunc(void);

void printme(void) {
        printf("before you go!\n");
}

int
main(void) {
        Exitfunc *ex = &printme;
        atexit(ex);
        return 0;
}
```
# 7.6 
# If we allocate an array of longs using calloc, is the array initialized to 0? If we allocate an array of pointers using calloc, is the array initialized to null pointers?

Yes. Yes. Respectively.

```c
#include <stdio.h>
#include <stdlib.h>
int
main(void) {
    long *larray;
    long **parray;
    int i = 0;
    larray = (long *)calloc(10, sizeof(long));
    for (i=0; i<10 ; i++){
        printf("%ld ",larray[i]);
    }

    printf("\n");
    parray  = calloc(10,sizeof(long*) );

    for (i=0; i<10; i++){
        printf("0x%08X ", parray[i]);
    }
    printf("\n");

    return 0;
}
```
```
vagrant@precise64:/vagrant/advC$ ./a.out
0 0 0 0 0 0 0 0 0 0
0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000
```
# 7.7 
# In the output from the size command at the end of Section 7.6, why aren’t any sizes given for the heap and the stack?
Heaps and Stacks are created at runtime. ```Size command``` examines the static binary file.

# 7.8 
# In Section 7.7, the two file sizes (879443 and 8378) don’t equal the sums of their respective text and data sizes. Why?
Executable contain more than just the text and data segments like the symbol table/debug information etc. 
```strip``` command helps eliminate the symbol table and other part of thr binary. 
**[github issue](https://github.com/maitreyak/APUE_Book_Solutions/issues/3)***

# 7.9
# In Section 7.7, why does the size of the executable file differ so dramatically when we use shared libraries for such a trivial program?
Shared libaries are large. By enabling static linking, we include all the libs in the executable. Hence the dramatic differece in size between statically linked and dynamically linked executables. ```file``` command gives you the linked details of a binary.
```
vagrant@precise64:/vagrant/advC$ gcc --static helloworld.c
vagrant@precise64:/vagrant/advC$ file a.out
a.out: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.24, BuildID[sha1]=0xd348976e691af2666b269d5768c81065da642853, not stripped
vagrant@precise64:/vagrant/advC$ gcc helloworld.c
vagrant@precise64:/vagrant/advC$ file a.out
a.out: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=0x431147a89cb77a58473cb310a19e658d548f7e9e, not stripped
```
**[github issue](https://github.com/maitreyak/APUE_Book_Solutions/issues/4)***

# 7.10 
# At the end of Section 7.10, we showed how a function can’t return a pointer to an automatic variable. Is the following code correct?
```c 
   1 #include <stdio.h>
  2
  3 int f1(){
  4     int val = 0;
  5     int *ptr = &val;
  6     while (val == 0) { <--- minor change.
  7         int val = 5; 
  8         ptr = &val;
  9         break; <--- minor change.
 10     }
 11     return (*ptr + 1);
 12 }
 13
 14 int
 15 main(void) {
 16     printf("%d", f1());
 17     return 0;
 18 }
```
The program dereferences address that of the deallocated ```if block local``` variable. The operation is dangerous. The program incorrect. 
```
(gdb) break stack.c:10
(gdb) break 9
Breakpoint 1 at 0x40051e: file stack.c, line 9.
(gdb) run
Starting program: /vagrant/advC/a.out
Breakpoint 1, f1 () at stack.c:9
9			break;
(gdb) inf local
val = 5
val = 0
ptr = 0x7fffffffe47c
(gdb) break 11
Breakpoint 2 at 0x40051f: file stack.c, line 11.
(gdb) continue
Continuing.
Breakpoint 2, f1 () at stack.c:11
11		return (*ptr + 1);
(gdb) inf local
val = 0
ptr = 0x7fffffffe47c
```
As you can see ptr still points to location that has been deallocated (or scheduled to be.). Hence the program is dangerous.

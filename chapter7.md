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

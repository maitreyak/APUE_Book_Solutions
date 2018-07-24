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
After the exit() call and before the _exit() call.

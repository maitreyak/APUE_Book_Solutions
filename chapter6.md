# 6.1 
# If the system uses a shadow file and we need to obtain the encrypted password, how do we do so?
With root access one could access the ```/etc/shadow``` file to take a look. A sample entry would be like below, username followed by the encrypted password delimited by ":" followed by password aging fields.  
```
vagrant:$6$aqzOtgCM$OxgoMP4JoqMJ1U1F3MZPo2iBefDRnRCXSfgIM36E5cfMNcE7GcNtH1P/tTC2QY3sX3BxxJ7r/9ciScIVTa55l0:15597:0:99999:7::
```

# 6.2 
# If you have superuser access and your system uses shadow passwords, implement the previous exercise.
Below is the program that prints out the encrypted password for the user ```vagrant```. As you can see the string matches the output from the previous excercise.
```c
#include <stdio.h>
#include <stdlib.h>
#include <shadow.h>

int
main(void) {
    struct spwd *shadow;
    if((shadow = getspnam("vagrant")) == NULL) {
        perror("shadow file error:");
        exit(-1);
    }
    printf("Encrypted password %s\n", shadow->sp_pwdp);

    return 0;
}
```
```
root@precise64:/vagrant/advC# gcc shadowpass.c
root@precise64:/vagrant/advC# ./a.out
Encrypted password $6$aqzOtgCM$OxgoMP4JoqMJ1U1F3MZPo2iBefDRnRCXSfgIM36E5cfMNcE7GcNtH1P/tTC2QY3sX3BxxJ7r/9ciScIVTa55l0
```
# 6.3 
# Write a program that calls uname and prints all the fields in the utsname structure. Compare the output to the output from the uname(1) command.
```c
#include <sys/utsname.h>
#include <stdio.h>

int
main(int argc, char *argv[]) {
    struct utsname unm;
    if (uname(&unm) < 0) {
        perror("uname error");
        return -1;
    }

    printf("sysname:  %s\n", unm.sysname);
    printf("nodename: %s\n", unm.nodename);
    printf("release:  %s\n", unm.release);
    printf("version:  %s\n", unm.version);
    printf("machine:  %s\n", unm.machine);
    return 0;
}
```
```
root@precise64:/vagrant/advC# ./a.out
sysname:  Linux
nodename: precise64
release:  3.2.0-23-generic
version:  #36-Ubuntu SMP Tue Apr 10 20:39:51 UTC 2012
machine:  x86_64
```
Comparing with the command. ```uname -a``` the output is nearly the same.
```
root@precise64:/vagrant/advC# uname -a
Linux precise64 3.2.0-23-generic #36-Ubuntu SMP Tue Apr 10 20:39:51 UTC 2012 x86_64 x86_64 x86_64 GNU/Linux
```
# 6.4 
# Calculate the latest time that can be represented by the time_t data type. After it wraps around, what happens?
```c
#include <stdio.h>
#include <time.h>
int
main(void){
    time_t t;
    printf("size of time_t in bytes %ld\n", sizeof(time_t));
    return 0;
}
```
To find the data type time_t encapsulates on this machine. We use the ```gcc -E``` option.
```
root@precise64:/vagrant/advC# gcc -E time_t.c | grep time_t
typedef long int __time_t;
typedef __time_t time_t;
```
Notice that time_t infact is a ```signed long``` (on this perticular node).
Let's run the program and find out the size of ```long```.
```
root@precise64:/vagrant/advC# ./a.out
size of time_t in bytes 8
```
The largest positive values that can stored would be ```2^63 - 1```. An overflow would yeild negative values, most modern system detech this.  
# 6.5
# Write a program to obtain the current time and print it using strftime, so that it looks like the default output from date(1). Set the TZ environment variable to different values and see what happens.
```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main(void)
{
    time_t     now;
    struct tm *ts;
    char       buf[80];

    setenv("TZ", "Australia/Brisbane", 1);
    tzset();
    time(&now);

    ts = localtime(&now);
    strftime(buf, sizeof(buf), "%a %Y-%m-%d %H:%M:%S %Z", ts);
    puts(buf);

    return 0;
}
```
The above program outputs the time for ```Australia/Brisbane``` in the ```date(1)``` command format. 
```
➜  advC ./a.out
Tue 2018-07-17 15:31:22 AEST
```
The date outout should be on my machines local timezone CDT. (Yikes now you know where I live. Sort of.)
```
➜  advC date
Tue Jul 17 00:32:38 CDT 2018
```

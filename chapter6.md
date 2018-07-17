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

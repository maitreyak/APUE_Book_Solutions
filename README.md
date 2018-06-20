# APUE_Book_Solutions
Solutions to exercises Advanced Programming in the unix environment 

# 3.1
# When reading or writing a disk file, are the functions described in this chapter really unbuffered? Explain.
The unbuffered read and write systems functions (not system calls mind you) do not use buffer in the user mode. However, their system call counterparts in the kernal mode do. Buffering could be in the form of page caching of files in memory, that are periodically flushed to disk. Therefore, the "unbuffered" functions described do use kernel level buffers. 

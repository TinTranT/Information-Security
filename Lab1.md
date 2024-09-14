#1. bof1.c

The goal is that we can execute secretFunc.

![Picture about bof1.c](/imgs/bof1.png)

From the program, the function gets(array) at line 11 has no limitation with the length of string, we can input a string with a length of more than 200 characters. Therefore, there is a buffer overflow vulnerability.

First of all, we compile a C program by using this command:

`gcc -g bof1.c -o bof1.out -fno-stack-protector -mpreferred-stack-boundary=2`

- "-fno-stack-protector" means we disable the stack protection mechanism.

- "-mpreferred-stack-boundary=2" sets the stack alignment to 2^2, or 4 bytes.

Stack frame of this program:

![Picture about stack frame of bof1.c](/imgs/Stack-Frame-bof1.png)

If we want to exploit this program, we must enter 204 characters + 4 bytes address to overwrite the original return address.

By using gdb, we learn that the address of secretFunc is 0804846b.

![Picture about using gdb to take address of secretFunc](/imgs/gdb-bof1.png)

Finally, we use `echo $(python -c "print('a'*204 + '\x6b\x84\x04\x08')") | ./bof1.out`.

- `echo` is used to display a line of text or output data

![Picture about a result of exploiting successfully bof1](/imgs/result-bof1.png)


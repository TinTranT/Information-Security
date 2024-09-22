# 1. bof1.c

The goal is that we can execute secretFunc.

![Picture about bof1.c](/Chapter-2/imgs/bof1/bof1.png)

From the program, the function gets(array) at line 11 has no limitation with the length of string, we can input a string with a length of more than 200 characters. Therefore, there is a buffer overflow vulnerability.

First of all, we compile a C program by using this command:

`gcc -g bof1.c -o bof1.out -fno-stack-protector -mpreferred-stack-boundary=2`

- "-fno-stack-protector" means we disable the stack protection mechanism.

- "-mpreferred-stack-boundary=2" sets the stack alignment to 2^2, or 4 bytes.

Stack frame of this program:

![Picture about stack frame of bof1.c](/Chapter-2/imgs/bof1/bof1-Stack-Frame.png)

If we want to exploit this program, we must enter 204 characters + 4 bytes address to overwrite the original return address.

By using gdb, we learn that the address of secretFunc is 0804846b.

![Picture about using gdb to take address of secretFunc](/Chapter-2/imgs/bof1/bof1-gdb.png)


Finally, we use `echo $(python -c "print('a'*204 + '\x6b\x84\x04\x08')") | ./bof1.out`.

- `echo` is used to display a line of text or output data

![Picture about a result of exploiting successfully bof1](/Chapter-2/imgs/bof1/bof1-result.png)

# 2. bof2.c

The goal is that check's value is 0xdeadbeef.

![Picture about bof2.c](/Chapter-2/imgs/bof2/bof2.png)

The function fgets(buf, 45, stdin) at line 10 can get a string of 45 characters but buf's length is only 40 characters. Therefore, we can exploit this vulnerability.

Stack frame of this program:

![Picture about stack frame of bof2.c](/Chapter-2/imgs/bof2/bof2-Stack-Frame.png)

We enter a 40 characters and 4 bytes to overwrite check's value.

`echo $(python -c "print('a'*40 + '\xef\xbe\xad\xde')") | ./bof2.out`

![Picture about replacing value of check](/Chapter-2/imgs/bof2/bof2-result-1.png)

![Picture about replacing value of check](/Chapter-2/imgs/bof2/bof2-result-2.png)

# 3. bof3.c

The goal is that the program execute shell.

![Picture about bof3.c](/Chapter-2/imgs/bof3/bof3.png)

The function fgets(buf, 133, stdin) at line 21 can get a string of 133 characters but buf's length is only 128 characters. Therefore, we can exploit this vulnerability.

Stack frame of this program:

![Picture about stack frame of bof3.c](/Chapter-2/imgs/bof3/bof3-Stack-Frame.png)

To get the address of shell, we use gdb and learn that the value is 0x0804845b

![Picture about using gdb to get address of shell](/Chapter-2/imgs/bof3/bof3-gdb.png)

or we can use `objdump -d bof3.out | grep shell`, it is more convenient.

After that, we use `echo $(python -c "print('a'*128 + '\x5b\x84\x04\x08')") | ./bof3.out`

![Picture about executing successfully function shell](/Chapter-2/imgs/bof3/bof3-result.png)

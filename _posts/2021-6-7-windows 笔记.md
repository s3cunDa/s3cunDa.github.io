# windows 笔记

## pe

由coff发展而来，32位称之为pe32，64位称之为pe+或者pe32+。

pe文件与elf文件差不多，也分为几类具体来说如下表：

| 种类   | 扩展名             |
| ------ | ------------------ |
| 可执行 | exe，scr           |
| 库     | dll，ocx，cpl，drv |
| 驱动   | sys，vxd           |
| 对象   | obj                |

pe文件格式如下图：

![pe文件格式](https://s3cunda.github.io/assets/post/pe文件格式.png)

![image-20210714161235160](https://s3cunda.github.io/assets/post/image-20210714161235160.png)

头部信息相对于elf文件内容更多一些，从cos头到节区头都是头部信息，映射关系与elf大同小异。

1.dos头：主要为了向后兼容，大小为64字节，两个重要字节：e_magic:dos签名，e_lfanew：指示nt头的偏移

```c
IMAGE_DOS_HEADER {
    WORD   e_magic;                // +0000h   -   EXE标志，“MZ”
    WORD   e_cblp;                 // +0002h   -   最后（部分）页中的字节数
    WORD   e_cp;                   // +0004h   -   文件中的全部和部分页数
    WORD   e_crlc;                 // +0006h   -   重定位表中的指针数
    WORD   e_cparhdr;              // +0008h   -   头部尺寸，以段落为单位
    WORD   e_minalloc;             // +000ah   -   所需的最小附加段
    WORD   e_maxalloc;             // +000ch   -   所需的最大附加段
    WORD   e_ss;                   // +000eh   -   初始的SS值（相对偏移量）
    WORD   e_sp;                   // +0010h   -   初始的SP值
    WORD   e_csum;                 // +0012h   -   补码校验值
    WORD   e_ip;                   // +0014h   -   初始的IP值
    WORD   e_cs;                   // +0016h   -   初始的CS值
    WORD   e_lfarlc;               // +0018h   -   重定位表的字节偏移量
    WORD   e_ovno;                 // +001ah   -   覆盖号
    WORD   e_res[4];               // +001ch   -   保留字00
    WORD   e_oemid;                // +0024h   -   OEM标识符
    WORD   e_oeminfo;              // +0026h   -   OEM信息
    WORD   e_res2[10];             // +0028h   -   保留字
    LONG   e_lfanew;               // +003ch   -   PE头相对于文件的偏移地址
  }
```



2.dos存根：stub，位于dos下方，可选且大小不固定，代码数据混合,整个DOS Stub是一个字节块，其内容随着链接时使用的链接器不同而不同.

3.nt头：结构体为IMAGE_NT_HEADERS,大小是f8，三个成员组成：

```c
IMAGE_NT_HEADERS {
    DWORD Signature;                      // +0000h   -   PE文件标识，“PE00”
    IMAGE_FILE_HEADER FileHeader;                   // +0004h   -   PE标准头
    IMAGE_OPTIONAL_HEADER32 OptionalHeader;         // +0018h   -   PE扩展头
}
```

​	签名结构体：值为0x50450000（PE00）

​	文件头，表示文件大致属性，结构体为IMAGE_FILEE_HEADER,有四个重要成员：

```c
IMAGE_FILE_HEADER {
    WORD    Machine;                             // +0004h   -   运行平台
    WORD    NumberOfSections;                    // +0006h   -   PE中节的数量
    DWORD   TimeDateStamp;                       // +0008h   -   文件创建日期和时间
    DWORD   PointerToSymbolTable;                // +000ch   -   指向符号表
    DWORD   NumberOfSymbols;                     // +0010h   -   符号表中的符号数量
    WORD    SizeOfOptionalHeader;                // +0014h   -   扩展头结构的长度
    WORD    Characteristics;                     // +0016h   -   文件属性

}
```



​		Machine：每个CPU都拥有唯一的Machine码，简容32位intel x86芯片的Machine码为14c。

​		NumberOfSections：指出文件中的节区数量

​		SizeOfOptionalHeader：指出结构体IMAGE_OPTIONAL_GEADER32的长度（32位系统）

​		Characteriss：表示文件属性，是否可运行，是否为dll等

​	可选头，结构体为IMAGE_OPRTIONAL_HEADER32，九个重要成员：

```c
IMAGE_OPTIONAL_HEADER {
    WORD    Magic;                                 // +0018h   -   魔术字107h = ROM Image，10bh = exe Image
    BYTE    MajorLinkerVersion;                    // +001ah   -   链接器版本号
    BYTE    MinorLinkerVersion;                    // +001bh   -   
    DWORD   SizeOfCode;                            // +001ch   -   所有含代码的节的总大小
    DWORD   SizeOfInitializedData;                 // +0020h   -   所有含已初始化数据的节的总大小
    DWORD   SizeOfUninitializedData;               // +0024h   -   所有含未初始化数据的节的大小
    DWORD   AddressOfEntryPoint;                   // +0028h   -   程序执行入口RVA
    DWORD   BaseOfCode;                            // +002ch   -   代码的节的起始RVA
    DWORD   BaseOfData;                            // +0030h   -   数据的节的起始RVA
    DWORD   ImageBase;                             // +0034h   -   程序的建议装载地址
    DWORD   SectionAlignment;                      // +0038h   -   内存中的节的对齐粒度
    DWORD   FileAlignment;                         // +003ch   -   文件中的节的对齐粒度
    WORD    MajorOperatingSystemVersion;           // +0040h   -   操作系统版本号
    WORD    MinorOperatingSystemVersion;           // +0042h   -   
    WORD    MajorImageVersion;                     // +0044h   -   该PE的版本号
    WORD    MinorImageVersion;                     // +0046h   -   
    WORD    MajorSubsystemVersion;                 // +0048h   -   所需子系统的版本号
    WORD    MinorSubsystemVersion;                 // +004ah   -   
    DWORD   Win32VersionValue;                     // +004ch   -   未用
    DWORD   SizeOfImage;                           // +0050h   -   内存中的整个PE映象尺寸
    DWORD   SizeOfHeaders;                         // +0054h   -   所有头+节表的大小
    DWORD   CheckSum;                              // +0058h   -   校验和
    WORD    Subsystem;                             // +005ch   -   文件的子系统
    WORD    DllCharacteristics;                    // +005eh   -   DLL文件特性
    DWORD   SizeOfStackReserve;                    // +0060h   -   初始化时的栈大小
    DWORD   SizeOfStackCommit;                     // +0064h   -   初始化时实际提交的栈大小
    DWORD   SizeOfHeapReserve;                     // +0068h   -   初始化时保留的堆大小
    DWORD   SizeOfHeapCommit;                      // +006ch   -   初始化时实际提交的堆大小
    DWORD   LoaderFlags;                           // +0070h   -   与调试有关
    DWORD   NumberOfRvaAndSizes;                   // +0074h   -   下面的数据目录结构的项目数量
    IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];     // 0078h   -   数据目录
}
```



​		magic：32为10B，64为20B

​		AddressOfEntryPoint：保存RVA值，指出程序最先执行的代码起始地址

​		ImageBase：指出文件的优先装入地址

​		SectionAlignment，FileAlignment：前者制定节区在内存中最小单位，后者代表磁盘中的最小单位。

​		SizeOfHeaders：整个pe文件的大小

​		SubSystem：区分系统驱动文件和可执行文件

​		NUmberOfRvaAndSize：指定datadirectory数组个数

​		Datadirectory：由IMAGE_DATA_DIRECTORY结构体组成的数组

```c
IMAGE_DATA_DIRECTORY {
    DWORD   VirtualAddress;                 // +0000h   -   数据的起始RVA
    DWORD   Size;                           // +0004h   -   数据块的长度
}
```

- 总的数据目录一共由16个相同的IMAGE_DATA_DIRECTORY结构连续排列在一起组成。
- 如果想在PE文件中寻找特定类型的数据，就需要从该结构开始。
- 这16个元组的数组每一项均代表PE中的某一个类型的数据，各数据类型为:

**数组编号中文描述**0导出表地址和大小1导入表地址和大小2资源表地址和大小3异常表地址和大小4属性证书数据地址和大小5基地址重定位表地址和大小6调试信息地址和大小7预留为08指向全局指针寄存器的值9线程局部存储地址和大小10加载配置表地址和大小11绑定导入表地址和大小12导入函数地址表地址和大小13延迟导入表地址和大小14CLR运行时头部数据地址和大小15系统保留



4.节区头

​	节区头中定义了各个节区的属性，包括不同的特性、访问权限等等，结构体为IMAGE_SECTION_HEADER，五个重要成员：

​	Virtualsize：内存中节区所占大小

​	VirtualAddress：内存中节区起始地址（RVA）

​	SizeOfRawData：磁盘中文件各节区所占大小

​	Characteristic：节区属性

如何映射？1.查找RVA所在的节区

2.**RAW - PointerToRawData = RVA - ImageBase**

**RAW = RVA - ImageBase + PointerToRawData**

example：ImageBase为0x10000000，节区为.text，文件中起始地址为0x00000400，内存中的起始地址为0x01001000，RVA = 5000，RAW = 5000 - 1000 + 400 = 4400。

地址：

- 虚拟内存地址（Virtual Address, VA）PE文件中的指令被装入内存后的地址。

- 相对虚拟内存地址（Reverse Virtual Address, RVA相对虚拟地址是内存地址相对于映射基址的偏移量。

- 文件偏移地址（File Offset Address, FOA）数据在PE文件中的地址叫文件偏移地址，这是文件在磁盘上存放时相对于文件开头的偏移。

- 装在基址（Image base）PE装入内存时的基地址。默认情况下，EXE文件在内存中的基地址时0x00400000, DLL文件是0x10000000。这些位置可以通过修改编译选项更改。

- 虚拟内存地址、映射基址、相对虚拟内存地址的关系：

- `va = imagebase + rva`

- 文件偏移是相对于文件开始处0字节的偏移，相对虚拟地址则是相对于装载基址0x00400000处的偏移。（1）PE文件中的数据按照磁盘数据标准存放，以0x200字节为基本单位进行组织，PE数据节的大小永远是0x200的整数倍。（2）当代码装入内存后，将按照内存数据标准存放，并以0x1000字节为基本单位进行组织，内存中的节总是0x1000的整数倍。

- 内存中数据节相对于装载基址的偏移量和文件中数据节的偏移量的差异称为节偏移。

- ```text
  文件偏移地址 = 虚拟内存地址（VA） - 装载基址（Image Base） - 节偏移 
               = RVA - 节偏移
  ```


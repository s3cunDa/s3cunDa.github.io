# xman re部分writeup

标签（空格分隔）： re
---

***1.re0***
题目开头的代码可以看出对judge函数进行了加密


        for ( i = 0; i <= 181; ++i )
      {
        envp = (const char **)(*((unsigned __int8 *)judge + i) ^ 0xCu);
        *((_BYTE *)judge + i) ^= 0xCu;
      }

在data段看到加密后的judge函数为乱码
可以看到judge函数的开头为0x600b00结尾为0x600bb5
但是在函数定义阶段定义的结尾为0x600b05
右键judge函数将其结尾改为0x600bb5
输入python脚本，将其解密（异或0xc）

    judge=0x600B00
    for i in range(182):
        addr=0x600B00+i
        byte=get_bytes(addr,1)
        byte=ord(byte)^0xC
        patch_byte(addr,byte)
        
解密后反编译，发现ida反编译后代码较混乱
回到textview视图，将judge函数取消定义，而后重新定义，可以看到judge函数的反编译代码：

    __int64 __fastcall judge(__int64 a1)
    {
      char v2; // [rsp+8h] [rbp-20h]
      char v3; // [rsp+9h] [rbp-1Fh]
      char v4; // [rsp+Ah] [rbp-1Eh]
      char v5; // [rsp+Bh] [rbp-1Dh]
      char v6; // [rsp+Ch] [rbp-1Ch]
      char v7; // [rsp+Dh] [rbp-1Bh]
      char v8; // [rsp+Eh] [rbp-1Ah]
      char v9; // [rsp+Fh] [rbp-19h]
      char v10; // [rsp+10h] [rbp-18h]
      char v11; // [rsp+11h] [rbp-17h]
      char v12; // [rsp+12h] [rbp-16h]
      char v13; // [rsp+13h] [rbp-15h]
      char v14; // [rsp+14h] [rbp-14h]
      char v15; // [rsp+15h] [rbp-13h]
      int i; // [rsp+24h] [rbp-4h]
    
      v2 = 102;
      v3 = 109;
      v4 = 99;
      v5 = 100;
      v6 = 127;
      v7 = 107;
      v8 = 55;
      v9 = 100;
      v10 = 59;
      v11 = 86;
      v12 = 96;
      v13 = 59;
      v14 = 110;
      v15 = 112;
      for ( i = 0; i <= 13; ++i )
        *(_BYTE *)(i + a1) ^= i;
      for ( i = 0; i <= 13 && *(_BYTE *)(i + a1) == *(&v2 + i); ++i )
        ;
      return nullsub_1();
    }

将v2至v5转换为字符类型，可得字符串：
fmcd\x7fk7d;V\x60;np
其中\x7f与\x60为不可显示字符（对应v6 127以及 v12 96）
可以看到后续运算将其与其下标进行异或运算
可写出解密脚本：

    flag_enc="fmcd\x7fk7d;V\x60;np"
    flag=""
    for i in range(len(flag_enc)):
        c=flag_enc[i]
        flag+=chr(ord(c)^i)
    
    print flag
    
得到flag：
flag{n1c3_j0b}
***2.re_1***
后缀名是.exe，运行发现是注册框程序
对MessageBoxA函数进行xref查询，可以找到关键代码段在0x401720调用messagebox函数，对此函数xref操作，定位至0x401621，向前查阅代码，可以看到在4015e0段判断输入的字符长度是否等于21h

    .text:004015E0                 push    ecx
    .text:004015E1                 push    esi
    .text:004015E2                 mov     esi, ecx
    .text:004015E4                 push    1
    .text:004015E6                 call    ?UpdateData@CWnd@@QAEHH@Z ; CWnd::UpdateData(int)
    .text:004015EB                 mov     ecx, [esi+294h]
    .text:004015F1                 lea     eax, [esi+294h]
    .text:004015F7                 cmp     dword ptr [ecx-8], 21h
    .text:004015FB                 jnz     short loc_40161F
    .text:004015FD                 push    ecx
    .text:004015FE                 mov     ecx, esp
    .text:00401600                 mov     [esp+8], esp
    .text:00401604                 push    eax
    .text:00401605                 call    ??0CString@@QAE@ABV0@@Z ; CString::CString(CString const &)
    .text:0040160A                 mov     ecx, esi
    .text:0040160C                 call    sub_401630
    .text:00401611                 test    al, al
    .text:00401613                 jz      short loc_40161F
    .text:00401615                 mov     ecx, esi
    .text:00401617                 call    sub_4016E0
    .text:0040161C                 pop     esi
    .text:0040161D                 pop     ecx
    .text:0040161E                 retn
    .text:0040161F ; ---------------------------------------------------------------------------
    .text:0040161F
    .text:0040161F loc_40161F:                             ; CODE XREF: .text:004015FB↑j
    .text:0040161F                                         ; .text:00401613↑j
    .text:0040161F                 mov     ecx, esi
    .text:00401621                 call    sub_401720
    .text:00401626                 pop     esi
    .text:00401627                 pop     ecx
    .text:00401628                 retn
有以上代码可以得到在字符串长度为21h时会进行下一步判断，即调用0x401630处函数，其汇编代码如下：

    .text:00401630 sub_401630      proc near               ; CODE XREF: .text:0040160C↑p
    .text:00401630
    .text:00401630 var_1           = byte ptr -1
    .text:00401630 arg_0           = dword ptr  4
    .text:00401630
    .text:00401630                 push    ecx
    .text:00401631                 push    ebx
    .text:00401632                 mov     ebx, ds:srand
    .text:00401638                 push    ebp
    .text:00401639                 push    esi
    .text:0040163A                 push    edi
    .text:0040163B                 xor     edi, edi
    .text:0040163D                 mov     ebp, ecx
    .text:0040163F                 mov     [esp+14h+var_1], 1
    .text:00401644                 mov     edx, 0Ah
    .text:00401649                 xor     esi, esi
    .text:0040164B
    .text:0040164B loc_40164B:                             ; CODE XREF: sub_401630+4E↓j
    .text:0040164B                 push    edx             ; Seed
    .text:0040164C                 call    ebx ; srand
    .text:0040164E                 add     esp, 4
    .text:00401651                 call    ds:rand
    .text:00401657                 cdq
    .text:00401658                 mov     ecx, 0Ah
    .text:0040165D                 idiv    ecx
    .text:0040165F                 mov     ecx, [esp+14h+arg_0]
    .text:00401663                 mov     cl, [edi+ecx]
    .text:00401666                 lea     eax, [esi+edx]
    .text:00401669                 cmp     cl, [eax+ebp+60h]
    .text:0040166D                 jz      short loc_401674
    .text:0040166F                 mov     [esp+14h+var_1], 0
    .text:00401674
    .text:00401674 loc_401674:                             ; CODE XREF: sub_401630+3D↑j
    .text:00401674                 add     esi, 0Ah
    .text:00401677                 inc     edi
    .text:00401678                 cmp     esi, 14Ah
    .text:0040167E                 jl      short loc_40164B
    .text:00401680                 lea     ecx, [esp+14h+arg_0] ; this
    .text:00401684                 call    ??1CString@@QAE@XZ ; CString::~CString(void)
    .text:00401689                 mov     al, [esp+14h+var_1]
    .text:0040168D                 pop     edi
    .text:0040168E                 pop     esi
    .text:0040168F                 pop     ebp
    .text:00401690                 pop     ebx
    .text:00401691                 pop     ecx
    .text:00401692                 retn    4
    .text:00401692 sub_401630      endp
    .text:00401692
可以观察到在0x401669处对输入的flag进行判断
进入od，在对应的代码段设置断点
输入长度为0x21的任意字符串，单步运行可爆破得flag：
flag{The-Y3ll0w-turb4ns-Upri$ing}
***3.evr***
拖入ida发现main函数不能反编译，显示栈指针有误，对于有误的代码地址alt+k将栈指针修复，得到反汇编后的main函数：

    __int64 __cdecl main_0()
    {
      int v0; // edx
      __int64 v1; // ST08_8
      int v3; // [esp-10h] [ebp-12Ch]
      signed int v4; // [esp-8h] [ebp-124h]
      signed int v5; // [esp-4h] [ebp-120h]
      bool v6; // [esp+Fh] [ebp-10Dh]
      char v7; // [esp+D7h] [ebp-45h]
      int i; // [esp+E0h] [ebp-3Ch]
      bool v9; // [esp+EFh] [ebp-2Dh]
      bool v10; // [esp+FBh] [ebp-21h]
      bool v11; // [esp+107h] [ebp-15h]
      HMODULE v12; // [esp+110h] [ebp-Ch]
      v12 = GetModuleHandleW(0);
      say((int)"Welcome to HCTF 2017\n\n");
      say((int)"Mark.09 is hijacking Shinji Ikari now...\n\n");
      say((int)"Check User: \n");
      v5 = 256;
      input((int)"%s", (unsigned int)Str);
      if ( !check_name() )
      {
        sub_8C114A();
        exit(0);
      }
      sub_8C11F9();
      say((int)"Check Start Code: \n");
      v4 = 128;
      input((int)"%s", (unsigned int)my_flag);
      while ( getchar() != 10 )
        ;
      if ( j_strlen(my_flag) != 35 )
      {
        sub_8C1398();
        sub_8C10FF();
        exit(0);
      }
      xor0x76((int)&my_flag1, (int)my_flag);
      v3 = dword_8CB780;
      sub_8C10EB((int)sub_8C105A, (int)sub_8C11EA);
      if ( sub_8C1361(1) )
      {
        sub_8C1398();
        sub_8C10FF();
        exit(0);
      }
      v6 = sub_8C10EB((int)sub_8C1023, (int)sub_8C139D) != 0;
      v11 = v6;
      sub_8C1023(byte_8CB5F0, &my_flag1);
      sub_8C1258(dword_8CB770, dword_8CB774, 204);
      if ( sub_8C1361(2) )
      {
        sub_8C1398();
        sub_8C10FF();
        exit(0);
      }
      v6 = sub_8C10EB((int)sub_8C106E, (int)sub_8C1046) != 0;
      v10 = v6;
      sub_8C106E(byte_8CB670, &my_flag1);
      sub_8C1258(dword_8CB778, dword_8CB77C, 205);
      if ( sub_8C1361(3) )
      {
        sub_8C1398();
        sub_8C10FF();
        exit(0);
      }
      v6 = sub_8C10EB((int)sub_8C105A, (int)sub_8C11EA) != 0;
      v9 = v6;
      sub_8C105A(byte_8CB6F0, &my_flag1);
      sub_8C1258(dword_8CB780, dword_8CB784, 221);
      for ( i = 0; i < 7; ++i )
      {
        byte_8CB577[i] = byte_8CB5F0[i];
        byte_8CB57E[i] = byte_8CB670[i];
        byte_8CB585[i] = byte_8CB6F0[i];
      }
      if ( sub_8C1447((int)&my_flag1, (int)&unk_8CB0DC) )
      {
        MessageBoxA(0, "> DETONATION FUNCTION\n    READY", "WILLE", 0);
        say((int)"[Y/N]?\n");
        input((int)"%c", (unsigned int)&v7);
        if ( v7 != 89 && v7 != 121 )
        {
          sub_8C1398();
          sub_8C10FF();
        }
        else
        {
          sub_8C1082();
          say((int)"Prevent IMPACT success\n");
        }
      }
      else
      {
        sub_8C1398();
        sub_8C10FF();
      }
      system("pause");
      HIDWORD(v1) = v0;
      LODWORD(v1) = 0;
      return v1;
    }
    
可以看到程序首先对输入的名字进行判断，成功后才会对输入的flag进行判断，但是名字的正确与否不会影响到flag的计算过程。
很明显的，flag长度为35
只对对于输入的flag判断的函数进行分析，可以看到首先将输入的flag异或0x76后存入另一个内存区域，而后对于其副本进行加解密操作。

    int __cdecl sub_8C1D40(int a1, int a2)
    {
      int result; // eax
      signed int i; // [esp+E8h] [ebp-8h]
    
      for ( i = 0; i < 7; ++i )
      {
        *(_BYTE *)(i + a1) = *(_BYTE *)(i + a2 + 7) ^ 0xAD;
        *(_BYTE *)(i + a1) = 2 * *(_BYTE *)(i + a1) & 0xAA | ((*(_BYTE *)(i + a1) & 0xAA) >> 1);
        result = i + 1;
      }
      return result;
    }
可以看到第一次将前七个字符加密运算
考虑最后的比较函数

     if ( sub_8C1447((int)&my_flag1, (int)&unk_8CB0DC) )
查看内存区域0x8cb0cc,将其转化为数组（d建）右键改变大小，shift+edump出内容，选择第五个参数，即c无符号字符型得到内容：
 30,  21,   2,  16,  13,  72,  72, 111, 221, 221, 
   72, 100,  99, 215,  46,  44, 254, 106, 109,  42, 
  242, 111, 154,  77, 139,  75,  10, 138,  79,  69, 
   23,  70,  79,  20,  11
而后的加密运算与第一次类似，不再赘述
最后将前几次的运算结果合并：

    for ( i = 0; i < 7; ++i )
      {
        byte_8CB577[i] = byte_8CB5F0[i];
        byte_8CB57E[i] = byte_8CB670[i];
        byte_8CB585[i] = byte_8CB6F0[i];
      }
由此可以得到逆向脚本算出flag

        flag_enc=[30,  21,   2,  16,  13,  72,  72, 111, 221, 221, 72, 100,  99, 215,  46,  44, 254, 106, 109,  42, 242, 111, 154,  77, 139,  75,  10, 138,  79,  69,  23,  70,  79,  20,  11];
    
    flag=""
    for i in range(7):
        flag+=chr(flag_enc[i]^0x76)
    for i in range(7):
        for c in range(0x20,0x7f):
            origc=c
            c=c^0x76^0xad
            c=((2*c)&0xff)&0xaa|(0xff&((c&0xaa)>>1))
            if c==flag_enc[7+i]:
                flag+=chr(origc)
                break
    for i in range(7):
        for c in range(0x20,0x7f):
            origc=c
            c=c^0x76^0xbe
            c=((4*c)&0xff)&0xcc|(0xff&((c&0xcc)>>2))
            if c==flag_enc[14+i]:
                flag+=chr(origc)
                break
    
    for i in range(7):
        for c in range(0x20,0x7f):
            origc=c
            c=c^0x76^0xef
            c=((16*c)&0xff)&0xf0|(0xff&((c&0xf0)>>4))
            if c==flag_enc[21+i]:
                flag+=chr(origc)
                break
    for i in range(7):
        flag+=chr(flag_enc[i+28]^0x76)
    print flag
得到flag：
hctf{>>D55_CH0CK3R_B0o0M!-9193a09b}
***4.simplecheck***


后缀名为apk，解压得到java文件，将classes.dex拖入java反编译器反编译得到源码：

    package com.a.simplecheck;
    
    public class a
    {
      private static int[] a = { 0, 146527998, 205327308, 94243885, 138810487, 408218567, 77866117, 71548549, 563255818, 559010506, 449018203, 576200653, 307283021, 467607947, 314806739, 341420795, 341420795, 469998524, 417733494, 342206934, 392460324, 382290309, 185532945, 364788505, 210058699, 198137551, 360748557, 440064477, 319861317, 676258995, 389214123, 829768461, 534844356, 427514172, 864054312 };
      private static int[] b = { 13710, 46393, 49151, 36900, 59564, 35883, 3517, 52957, 1509, 61207, 63274, 27694, 20932, 37997, 22069, 8438, 33995, 53298, 16908, 30902, 64602, 64028, 29629, 26537, 12026, 31610, 48639, 19968, 45654, 51972, 64956, 45293, 64752, 37108 };
      private static int[] c = { 38129, 57355, 22538, 47767, 8940, 4975, 27050, 56102, 21796, 41174, 63445, 53454, 28762, 59215, 16407, 64340, 37644, 59896, 41276, 25896, 27501, 38944, 37039, 38213, 61842, 43497, 9221, 9879, 14436, 60468, 19926, 47198, 8406, 64666 };
      private static int[] d = { 0, -341994984, -370404060, -257581614, -494024809, -135267265, 54930974, -155841406, 540422378, -107286502, -128056922, 265261633, 275964257, 119059597, 202392013, 283676377, 126284124, -68971076, 261217574, 197555158, -12893337, -10293675, 93868075, 121661845, 167461231, 123220255, 221507, 258914772, 180963987, 107841171, 41609001, 276531381, 169983906, 276158562 };
      
      public static boolean a(String paramString)
      {
        if (paramString.length() != b.length) {
          return false;
        }
        int[] arrayOfInt = new int[a.length];
        arrayOfInt[0] = 0;
        paramString = paramString.getBytes();
        int k = paramString.length;
        int i = 0;
        int j = 1;
        while (i < k)
        {
          arrayOfInt[j] = paramString[i];
          j += 1;
          i += 1;
        }
        i = 0;
        for (;;)
        {
          if (i >= c.length) {
            break label166;
          }
          if ((a[i] != b[i] * arrayOfInt[i] * arrayOfInt[i] + c[i] * arrayOfInt[i] + d[i]) || (a[(i + 1)] != b[i] * arrayOfInt[(i + 1)] * arrayOfInt[(i + 1)] + c[i] * arrayOfInt[(i + 1)] + d[i])) {
            break;
          }
          i += 1;
        }
        label166:
        return true;
      }
    }

首先判断输入参数是否长度与b长度相等，若不相等则返回错误。
而后将输入的参数复制到arrayOfInt中I（下标从1开始，未对齐）
而后进行一个一元二次方程组的运算
爆破可得flag：

    from string import printable
    
    a = [0, 146527998, 205327308, 94243885, 138810487, 408218567, 77866117, 71548549, 563255818, 559010506, 449018203, 576200653, 307283021, 467607947, 314806739, 341420795, 341420795, 469998524, 417733494, 342206934, 392460324, 382290309, 185532945, 364788505, 210058699, 198137551, 360748557, 440064477, 319861317, 676258995, 389214123, 829768461, 534844356, 427514172, 864054312]
    b = [13710, 46393, 49151, 36900, 59564, 35883, 3517, 52957, 1509, 61207, 63274, 27694, 20932, 37997, 22069, 8438, 33995, 53298, 16908, 30902, 64602, 64028, 29629, 26537, 12026, 31610, 48639, 19968, 45654, 51972, 64956, 45293, 64752, 37108]
    c = [38129, 57355, 22538, 47767, 8940, 4975, 27050, 56102, 21796, 41174, 63445, 53454, 28762, 59215, 16407, 64340, 37644, 59896, 41276, 25896, 27501, 38944, 37039, 38213, 61842, 43497, 9221, 9879, 14436, 60468, 19926, 47198, 8406, 64666]
    d = [0, -341994984, -370404060, -257581614, -494024809, -135267265, 54930974, -155841406, 540422378, -107286502, -128056922, 265261633, 275964257, 119059597, 202392013, 283676377, 126284124, -68971076, 261217574, 197555158, -12893337, -10293675, 93868075, 121661845, 167461231, 123220255, 221507, 258914772, 180963987, 107841171, 41609001, 276531381, 169983906, 276158562]
    flag = ''
    for i2 in range(len(c)):
        for i in printable:
            if (a[i2] == (((b[i2] * ord(i)) * ord(i)) + (c[i2] * ord(i))) + d[i2] ):
                flag += i
                break
    for i in printable:
        temp = len(c)
        if a[temp] == (((b[temp-1] * ord(i)) * ord(i)) + (c[temp-1] * ord(i))) + d[temp-1]:
            flag += i
            break
    print flag
flag：
flag{MAth_i&_GOOd_DON7_90V_7hInK?}
***5.hide***
文件加了壳，不能直接丢进ida，需要手动脱壳
命令如下：

   

     sudo dd if=/proc/\$(pidof hide)/mem of=hide_dump1 skip=4194304  bs=1c count=827392
    sudo dd if=/proc/$(pidof hide)/mem of=hide_dump2 skip=7110656  bs=1c count=20480
    cat hide_dump1 hide_dump2 >hide_dump

得到的hide_dump文件丢入ida可以观察到反编译后代码
查找字符串可以看到“enter the flag”被引用了两次
进入第一个函数查看

   

     if ( (unsigned int)sub_4009AE(&v6) != 0 )
      {
        v0 = "You are right\n";
        sub_43E9B0(1LL, "You are right\n", 14LL);
      }
查看函数4009ae

    _BOOL8 __fastcall sub_4009AE(__int64 a1)
    {
      return (unsigned int)sub_400360(a1, "qwb{this_is_wrong_flag}") == 0;
    }
输入flag测试发现真的是假的flag
于是找到第二个引用该字符串的函数

    LOAD:00000000004C8EC2                 mov     rsi, offset aEnterTheFlag ; "Enter the flag:\n"
    LOAD:00000000004C8EC9                 mov     rdx, 10h
    LOAD:00000000004C8ED0                 xor     eax, eax
    LOAD:00000000004C8ED2                 inc     eax
    LOAD:00000000004C8ED4                 syscall                 ; LINUX - sys_write
    LOAD:00000000004C8ED6                 xor     rdi, rdi
    LOAD:00000000004C8ED9                 xor     eax, eax
    LOAD:00000000004C8EDB                 mov     rsi, offset flag
    LOAD:00000000004C8EE2                 mov     rdx, 20h
    LOAD:00000000004C8EE9                 syscall                 ; LINUX - sys_read
    LOAD:00000000004C8EEB                 cmp     eax, 0
    LOAD:00000000004C8EEE                 jle     loc_4C8FA9
可以看到这里同样也引用了该字符串
初步猜测此处为正确的程序执行流
定义函数发现该处定义不了函数因为堆栈以及一些ida反编译因素
于是在00000000004C8EF4处定义函数
反编译代码如下：

    signed __int64 sub_4C8EF4()
    {
      _BYTE *v0; // rdi
      __int64 *v1; // rsi
      unsigned __int64 v2; // rdx
      signed __int64 result; // rax
    
      if ( strlen((const char *)&flag) == 21
        && BYTE1(flag) == 'w'
        && BYTE2(flag) == 'b'
        && BYTE3(flag) == '{'
        && *((_BYTE *)&flag + 20) == '}' )
      {
        xtea((__int64)&flag + 4);
        xor_index((__int64)&flag + 4);
        xtea((__int64)&flag + 4);
        xor_index((__int64)&flag + 4);
        xtea((__int64)&flag + 4);
        v0 = (char *)&flag + 4;
        xor_index((__int64)&flag + 4);
        v1 = byte_4C8CB0;
        v2 = 0LL;
        while ( v2 < 0x10 && *v0 == *(_BYTE *)v1 )
        {
          ++v2;
          ++v0;
          v1 = (__int64 *)((char *)v1 + 1);
        }
      }
      __asm { syscall; LINUX - sys_write }
      result = 60LL;
      __asm { syscall; LINUX - sys_exit }
      return result;
    }
可以看到其加密的方式，三次tea加密以及三次xor
解密脚本：

    #coding=utf-8
    import struct
    import string
    def u32(data):
        return struct.unpack("<I",data)[0]
     
    def p32(data):
        return struct.pack("<I",data)
     
    def u64(data):
        return struct.unpack("<Q",data)[0]
     
    def p64(data):
        return struct.pack("<Q",data)
     
    input1 = '1234567890123456'
    input1 = bytearray(input1)
    CONST_STR = 's1IpP3rEv3Ryd4Y3'
    CONST = 0X676E696C
    v4 = 0
    v4_4=0
    v4_4_arr = [0 for i in range(0,9)]
    for i in range(1,9):
        v4_4_arr[i] = (v4_4_arr[i-1]+CONST)&0XFFFFFFFF
     
    def re_block(byte_arr_8):
        i = 0
        v3 = u32(byte_arr_8[0:4])
        v4 = u32(byte_arr_8[4:8])
        print 'round', v3, v4
        for i in range(7,-1,-1):
            v30 = (v3 << 4) & 0xffffffff
            v2c = v3 >> 5
            edx = v30 ^ v2c
            v30 = (v3 + edx) & 0xffffffff
     
            v28 = (v4_4_arr[i+1] >> 11) & 3
            edx = u32(CONST_STR[v28 * 4:(v28 + 1) * 4])
            v2c = (v4_4_arr[i+1] + edx) & 0xffffffff  # xxxx
            # print 'v2c',hex(v2c)
            print v30 ^ v2c
            v4 = (v4+0x100000000-(v30 ^ v2c)) & 0xffffffff
     
            v30 = (v4 << 4) & 0xffffffff
            v2c = v4 >> 5
            edx = v30 ^ v2c
     
            v30 = (v4 + edx) & 0xffffffff
     
            v28 = v4_4_arr[i] & 3
            edx = u32(CONST_STR[v28 * 4:(v28 + 1) * 4])
            v2c = (v4_4_arr[i] + edx) & 0xffffffff
     
            v3 = (v3+0x100000000-(v30 ^ v2c)) & 0xffffffff
     
            print 'round',i,v3,v4
        byte_arr_8[0:4] = p32(v3)
        byte_arr_8[4:8] = p32(v4)
        return byte_arr_8
    def xor16(byte_arr_16):
        for i in range(0,16):
            byte_arr_16[i]^=i
    def re_all(str16):
        byte_arr = bytearray(str16)
        xor16(byte_arr)#传入整个bytearray，就是传入地址
        byte_arr[0:8] = re_block(byte_arr[0:8])#传入部分bytearray，就是复制之后再传入
        byte_arr[8:16] = re_block(byte_arr[8:16])
        xor16(byte_arr)
        byte_arr[0:8] = re_block(byte_arr[0:8])
        byte_arr[8:16] = re_block(byte_arr[8:16])
        xor16(byte_arr)
        byte_arr[0:8] = re_block(byte_arr[0:8])
        byte_arr[8:16] = re_block(byte_arr[8:16])
        return str(byte_arr)
     
     
    def block(str8):
        i=0
        input1=bytearray(str8)
        v3 = u32(input1[8 * i:8 * i + 4])
        v4 = u32(input1[8 * i + 4:8 * (i + 1)])
        v4_4 = 0  ##0000
        for j in range(0, 8):
     
            v30 = (v4 << 4) & 0xffffffff
            v2c = v4 >> 5
            edx = v30 ^ v2c
     
            v30 = (v4 + edx) & 0xffffffff
     
            v28 = v4_4 & 3
            edx = u32(CONST_STR[v28 * 4:(v28 + 1) * 4])
            v2c = (v4_4 + edx) & 0xffffffff
     
            v3 = ((v30 ^ v2c) + v3) & 0xffffffff
     
     
            v4_4 = (v4_4 + CONST) & 0xffffffff
     
            v30 = (v3 << 4) & 0xffffffff
            v2c = v3 >> 5
            edx = v30 ^ v2c
            v30 = (v3 + edx) & 0xffffffff
     
            v28 = (v4_4 >> 11) & 3
            edx = u32(CONST_STR[v28 * 4:(v28 + 1) * 4])
            v2c = (v4_4 + edx) & 0xffffffff  # xxxx
            print v30 ^ v2c
            v4 = ((v30 ^ v2c) + v4) & 0xffffffff
     
            print 'round', j, v3, v4
     
        input1[8 * i:8 * i + 4] = p32(v3)
        input1[8 * i + 4:8 * (i + 1)] = p32(v4)
        str8_1=str(input1)
        return str8_1
    def block2(str8):
        input1 = bytearray(str8)
        for i in range(0,8):
            input1[i]=input1[1]^i
        return str(input1)
    des = ('52B8137F358CF21B'+'F46386D2734F1E31').decode('hex')
    print len(des)
     
    print block('12345678').encode('hex')
    des1 = '5b90ef3f91b58fe6'.decode('hex')
    print re_all(bytearray(des))
***6.magic***
真的看不懂。。
[http://www.sohu.com/a/236355836_354899][1]
这里有一篇wp跟着下来还是不是很清楚


  [1]: http://www.sohu.com/a/236355836_354899
***7.re2***
丢进ida，32位程序
分析一下main函数

    int __cdecl main(int argc, const char **argv, const char **envp)
    {
      int result; // eax
      HANDLE v4; // eax
      DWORD NumberOfBytesWritten; // [esp+4h] [ebp-24h]
      char flag; // [esp+8h] [ebp-20h]
    
      say((int)aPleaseInputFla);
      scanf(a31s, &flag);
      if ( strlen(&flag) == 19 )
      {
        wat();
        v4 = CreateFileA(FileName, 0x40000000u, 0, 0, 2u, 0x80u, 0);
        WriteFile(v4, &flag, 0x13u, &NumberOfBytesWritten, 0);
        wrongflag_0(&flag, &NumberOfBytesWritten);
        if ( NumberOfBytesWritten == 1 )
          say((int)aRightFlagIsYou);
        else
          say((int)aWrong);
        system(aPause);
        result = 0;
      }
      else
      {
        say((int)aWrong);
        system(aPause);
        result = 0;
      }
      return result;
    }

逻辑很清晰，进入wrongflag函数：

    signed int __cdecl wrongflag_0(const char *flag, _DWORD *a2)
    {
      signed int result; // eax
      unsigned int v3; // kr04_4
      char v4[24]; // [esp+Ch] [ebp-18h]
    
      result = 0;
      strcpy(v4, "This_is_not_the_flag");
      v3 = strlen(flag) + 1;
      if ( (signed int)(v3 - 1) > 0 )
      {
        while ( v4[flag - v4 + result] == v4[result] )
        {
          if ( ++result >= (signed int)(v3 - 1) )
          {
            if ( result == 21 )
            {
              result = (signed int)a2;
              *a2 = 1;
            }
            return result;
          }
        }
      }
      return result;
    }
假的flag，常规套路
于是乎又回到main函数
发现在判断长度是否为19后调用了一个神奇的函数，初步考虑其为关键函数

    int wat()
    {
      HMODULE v0; // eax
      DWORD v2; // eax
    
      v2 = GetCurrentProcessId();
      hProcess = OpenProcess(0x1F0FFFu, 0, v2);
      v0 = LoadLibraryA(LibFileName);
      proc_addr = (int)GetProcAddress(v0, ProcName);
      lpAddress = (LPVOID)proc_addr;
      if ( !proc_addr )
        return say((int)aApi);
      unk_40C9B4 = *(_DWORD *)lpAddress;
      *((_BYTE *)&unk_40C9B4 + 4) = *((_BYTE *)lpAddress + 4);
      jmp_near = 0xE9u;
      dword_40C9BD = (char *)sub_401080 - (char *)lpAddress - 5;
      return sub_4010D0();
    }
本题可以看到在此函数里对于函数本题开启了写的保护权限，并且在函数后半段对写的指令以及地址进行了赋值，而后调用了sub4010d
将其命名为addr

    .data:0040C9BB                 db    0
    .data:0040C9BC jmp_near        db 0                    ; DATA XREF: sub_4010D0+3A↑o
    .data:0040C9BC                                         ; wat-2B↑w
    .data:0040C9BD addr            dd 0
    
        BOOL sub_4010D0()
    {
      DWORD v1; // [esp+4h] [ebp-8h]
      DWORD flOldProtect; // [esp+8h] [ebp-4h]
    
      v1 = 0;
      VirtualProtectEx(hProcess, proc_addr_2, 5u, 4u, &flOldProtect);
      WriteProcessMemory(hProcess, proc_addr_2, &jmp_near, 5u, 0);
      return VirtualProtectEx(hProcess, proc_addr_2, 5u, flOldProtect, &v1);
    }
返回之后main函数跳到hook上
进入401080函数，在addr处找到了关键函数suspect

    signed int __cdecl suspect(char *dst, signed int len)
    {
      char idx; // al
      char temp; // bl
      char v4; // cl
      int v5; // eax
    
      idx = 0;
      if ( len > 0 )
      {
        do
        {
          if ( idx == 18 )
          {
            dst[18] ^= 0x13u;
          }
          else
          {
            if ( idx % 2 )
              temp = dst[idx] - idx;
            else
              temp = dst[idx + 2];
            dst[idx] = idx ^ temp;
          }
          ++idx;
        }
        while ( idx < len );
      }
      v4 = 0;
      if ( len <= 0 )
        return 1;
      v5 = 0;
      while ( enc[v5] == dst[v5] )
      {
        v5 = ++v4;
        if ( v4 >= len )
          return 1;
      }
      return 0;
    }
最后的`while ( enc[v5] == dst[v5] )`可以看出enc为密文（0x40a030为其地址）
写出解密脚本：

    enc = [97, 106, 121, 103, 107, 70, 109, 46, 127, 95, 126, 45, 83, 86, 123, 56, 109, 76, 110,0]
    flag  = []
    enc[18] ^=0x13
    for i in range(19):
    	
    	if i==0:
    		pass
    	else:
    		
    		if i%2==1:
    			enc[i] ^= i
    			enc[i] += i
    		else:
    			if i == 18:
    				pass
    			else:
    				enc[i] ^= i
    
    	flag.append(chr(enc[i]))
    new = ''
    for j in range(19):
    	i = 18-j
    	if i%2 == 0 and i != 0:
    		flag[i] = flag[i-2]
    
    for i in flag:
    	new += i
    print new
运行得flag：
flag{Ho0k_w1th_Fun}（第一位无解，猜测得到f    lol）
ps：hook技巧并不会，经pizza大佬指点后豁然开朗
+++
title = "尝试读懂Go ASM"
date = 2024-01-28T22:53:59+08:00
lastmod = 2024-01-28T22:53:59+08:00
author = ["jiandong.liu93"]
tags = ["golang"]
draft = true
+++

# 问题

某同学在日常工作中写出了一段十分诡异的代码，在CR中产生的激烈讨论。

```go
var a = map[string]string{}

func main() {
    go func() {
        v := a["key1"]
        log.Println(v)
    }()

    go func() {
        b := make(map[string]string)
        a = b
    }()
}
```

显然，在工作中写出这种有明显争议的代码是十分不好的，但如何证明这段代码确实不会panic也确实是个问题。

# 思路

与其研究各种语法特性，不如借机研究一下Go的汇编，通过更底层的行为描述确认运行时的行为。

# 验证

以下代码解释参考了GPT的结果，AI改变世界~

Go汇编引入了4个伪寄存器，这4个寄存器时编译器用来维护上下文、特殊标识等作用的

- FP(Frame pointer):记录栈帧的初始地址，用于访问函数的参数与局部变量

- PC(Program counter):记录当前正在执行的指令，控制执行流程

- SB(Static base pointer):SB指向了可执行程序的数据段的开始。用于访问全局变量和函数地址。

- SP(Stack pointer):栈顶，用于管理调用栈

首先声明一个main方法

```
0x0000 00000 (main.go:5)  TEXT    main.main(SB), ABIInternal, $32-0
// TEXT 包名.方法名(SB 这是基于静态地址的数据), ABIInternal, $栈帧大小-参数及返回值大小
```

栈帧大小如何计算

```go
PCDATA  $0, $-2
CMP     R16, RSP
BLS     60          // Branch if Lower or Same
PCDATA  $0, $-1
```

一段费解的代码，由于CMP命令会产生标志位变化，但不会计算得到一个值，为了垃圾回收器和其他运行时组件能够正确地解释生成的代码，使用PCDATA命令插入一个标记。

```go
main.main STEXT size=80 args=0x0 locals=0x18 funcid=0x0 align=0x0
        0x0000 00000 (main.go:5)  TEXT    main.main(SB), ABIInternal, $32-0
        0x0000 00000 (main.go:5)  MOVD    16(g), R16
        0x0004 00004 (main.go:5)  PCDATA  $0, $-2
        0x0004 00004 (main.go:5)  CMP     R16, RSP
        0x0008 00008 (main.go:5)  BLS     60
        0x000c 00012 (main.go:5)  PCDATA  $0, $-1
        0x000c 00012 (main.go:5)  MOVD.W  R30, -32(RSP)
        0x0010 00016 (main.go:5)  MOVD    R29, -8(RSP)
        0x0014 00020 (main.go:5)  SUB     $8, RSP, R29
        0x0018 00024 (main.go:5)  FUNCDATA        $0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0018 00024 (main.go:5)  FUNCDATA        $1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0018 00024 (main.go:6)  MOVD    $main.main.func1·f(SB), R0
        0x0020 00032 (main.go:6)  PCDATA  $1, $0
        0x0020 00032 (main.go:6)  CALL    runtime.newproc(SB)
        0x0024 00036 (main.go:10) MOVD    $main.main.func2·f(SB), R0
        0x002c 00044 (main.go:10) CALL    runtime.newproc(SB)
        0x0030 00048 (main.go:14) LDP     -8(RSP), (R29, R30)
        0x0034 00052 (main.go:14) ADD     $32, RSP
        0x0038 00056 (main.go:14) RET     (R30)
        0x003c 00060 (main.go:14) NOP
        0x003c 00060 (main.go:5)  PCDATA  $1, $-1
        0x003c 00060 (main.go:5)  PCDATA  $0, $-2
        0x003c 00060 (main.go:5)  MOVD    R30, R3
        0x0040 00064 (main.go:5)  CALL    runtime.morestack_noctxt(SB)
        0x0044 00068 (main.go:5)  PCDATA  $0, $-1
        0x0044 00068 (main.go:5)  JMP     0
        0x0000 90 0b 40 f9 ff 63 30 eb a9 01 00 54 fe 0f 1e f8  ..@..c0....T....
        0x0010 fd 83 1f f8 fd 23 00 d1 00 00 00 90 00 00 00 91  .....#..........
        0x0020 00 00 00 94 00 00 00 90 00 00 00 91 00 00 00 94  ................
        0x0030 fd fb 7f a9 ff 83 00 91 c0 03 5f d6 e3 03 1e aa  .........._.....
        0x0040 00 00 00 94 ef ff ff 17 00 00 00 00 00 00 00 00  ................
        rel 24+8 t=3 main.main.func1·f+0
        rel 32+4 t=9 runtime.newproc+0
        rel 36+8 t=3 main.main.func2·f+0
        rel 44+4 t=9 runtime.newproc+0
        rel 64+4 t=9 runtime.morestack_noctxt+0
main.main.func1 STEXT size=80 args=0x0 locals=0x28 funcid=0x0 align=0x0
        0x0000 00000 (main.go:6)  TEXT    main.main.func1(SB), ABIInternal, $48-0
        0x0000 00000 (main.go:6)  MOVD    16(g), R16
        0x0004 00004 (main.go:6)  PCDATA  $0, $-2
        0x0004 00004 (main.go:6)  CMP     R16, RSP
        0x0008 00008 (main.go:6)  BLS     68
        0x000c 00012 (main.go:6)  PCDATA  $0, $-1
        0x000c 00012 (main.go:6)  MOVD.W  R30, -48(RSP)
        0x0010 00016 (main.go:6)  MOVD    R29, -8(RSP)
        0x0014 00020 (main.go:6)  SUB     $8, RSP, R29
        0x0018 00024 (main.go:6)  FUNCDATA        $0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0018 00024 (main.go:6)  FUNCDATA        $1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0018 00024 (main.go:6)  PCDATA  $0, $-3
        0x0018 00024 (main.go:7)  MOVD    main.a(SB), R1
        0x0020 00032 (main.go:7)  PCDATA  $0, $-1
        0x0020 00032 (main.go:7)  MOVD    $type:map[string]string(SB), R0
        0x0028 00040 (main.go:7)  MOVD    $go:string."key1"(SB), R2
        0x0030 00048 (main.go:7)  MOVD    $4, R3
        0x0034 00052 (main.go:7)  PCDATA  $1, $0
        0x0034 00052 (main.go:7)  CALL    runtime.mapaccess1_faststr(SB)
        0x0038 00056 (main.go:8)  LDP     -8(RSP), (R29, R30)
        0x003c 00060 (main.go:8)  ADD     $48, RSP
        0x0040 00064 (main.go:8)  RET     (R30)
        0x0044 00068 (main.go:8)  NOP
        0x0044 00068 (main.go:6)  PCDATA  $1, $-1
        0x0044 00068 (main.go:6)  PCDATA  $0, $-2
        0x0044 00068 (main.go:6)  MOVD    R30, R3
        0x0048 00072 (main.go:6)  CALL    runtime.morestack_noctxt(SB)
        0x004c 00076 (main.go:6)  PCDATA  $0, $-1
        0x004c 00076 (main.go:6)  JMP     0
        0x0000 90 0b 40 f9 ff 63 30 eb e9 01 00 54 fe 0f 1d f8  ..@..c0....T....
        0x0010 fd 83 1f f8 fd 23 00 d1 1b 00 00 90 61 03 40 f9  .....#......a.@.
        0x0020 00 00 00 90 00 00 00 91 02 00 00 90 42 00 00 91  ............B...
        0x0030 e3 03 7e b2 00 00 00 94 fd fb 7f a9 ff c3 00 91  ..~.............
        0x0040 c0 03 5f d6 e3 03 1e aa 00 00 00 94 ed ff ff 17  .._.............
        rel 24+8 t=41 main.a+0
        rel 32+8 t=3 type:map[string]string+0
        rel 40+8 t=3 go:string."key1"+0
        rel 52+4 t=9 runtime.mapaccess1_faststr+0
        rel 72+4 t=9 runtime.morestack_noctxt+0
main.main.func2 STEXT size=96 args=0x0 locals=0x8 funcid=0x0 align=0x0
        0x0000 00000 (main.go:10) TEXT    main.main.func2(SB), ABIInternal, $16-0
        0x0000 00000 (main.go:10) MOVD    16(g), R16
        0x0004 00004 (main.go:10) PCDATA  $0, $-2
        0x0004 00004 (main.go:10) CMP     R16, RSP
        0x0008 00008 (main.go:10) BLS     80
        0x000c 00012 (main.go:10) PCDATA  $0, $-1
        0x000c 00012 (main.go:10) MOVD.W  R30, -16(RSP)
        0x0010 00016 (main.go:10) MOVD    R29, -8(RSP)
        0x0014 00020 (main.go:10) SUB     $8, RSP, R29
        0x0018 00024 (main.go:10) FUNCDATA        $0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0018 00024 (main.go:10) FUNCDATA        $1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0018 00024 (main.go:11) PCDATA  $1, $0
        0x0018 00024 (main.go:11) CALL    runtime.makemap_small(SB)
        0x001c 00028 (main.go:12) PCDATA  $0, $-2
        0x001c 00028 (main.go:12) MOVWU   runtime.writeBarrier(SB), R1
        0x0024 00036 (main.go:12) CBZW    R1, 60
        0x0028 00040 (main.go:12) CALL    runtime.gcWriteBarrier2(SB)
        0x002c 00044 (main.go:12) MOVD    R0, (R25)
        0x0030 00048 (main.go:12) MOVD    main.a(SB), R1
        0x0038 00056 (main.go:12) MOVD    R1, 8(R25)
        0x003c 00060 (main.go:12) MOVD    R0, main.a(SB)
        0x0044 00068 (main.go:13) LDP     -8(RSP), (R29, R30)
        0x0048 00072 (main.go:13) ADD     $16, RSP
        0x004c 00076 (main.go:13) RET     (R30)
        0x0050 00080 (main.go:13) NOP
        0x0050 00080 (main.go:10) PCDATA  $1, $-1
        0x0050 00080 (main.go:10) PCDATA  $0, $-2
        0x0050 00080 (main.go:10) MOVD    R30, R3
        0x0054 00084 (main.go:10) CALL    runtime.morestack_noctxt(SB)
        0x0058 00088 (main.go:10) PCDATA  $0, $-1
        0x0058 00088 (main.go:10) JMP     0
        0x0000 90 0b 40 f9 ff 63 30 eb 49 02 00 54 fe 0f 1f f8  ..@..c0.I..T....
        0x0010 fd 83 1f f8 fd 23 00 d1 00 00 00 94 1b 00 00 90  .....#..........
        0x0020 61 03 40 b9 c1 00 00 34 00 00 00 94 20 03 00 f9  a.@....4.... ...
        0x0030 1b 00 00 90 61 03 40 f9 21 07 00 f9 1b 00 00 90  ....a.@.!.......
        0x0040 60 03 00 f9 fd fb 7f a9 ff 43 00 91 c0 03 5f d6  `........C...._.
        0x0050 e3 03 1e aa 00 00 00 94 ea ff ff 17 00 00 00 00  ................
        rel 24+4 t=9 runtime.makemap_small+0
        rel 28+8 t=40 runtime.writeBarrier+0
        rel 40+4 t=9 runtime.gcWriteBarrier2+0
        rel 48+8 t=41 main.a+0
        rel 60+8 t=41 main.a+0
        rel 84+4 t=9 runtime.morestack_noctxt+0
main.init STEXT size=96 args=0x0 locals=0x8 funcid=0x0 align=0x0
        0x0000 00000 (main.go:3)  TEXT    main.init(SB), PKGINIT|ABIInternal, $16-0
        0x0000 00000 (main.go:3)  MOVD    16(g), R16
        0x0004 00004 (main.go:3)  PCDATA  $0, $-2
        0x0004 00004 (main.go:3)  CMP     R16, RSP
        0x0008 00008 (main.go:3)  BLS     80
        0x000c 00012 (main.go:3)  PCDATA  $0, $-1
        0x000c 00012 (main.go:3)  MOVD.W  R30, -16(RSP)
        0x0010 00016 (main.go:3)  MOVD    R29, -8(RSP)
        0x0014 00020 (main.go:3)  SUB     $8, RSP, R29
        0x0018 00024 (main.go:3)  FUNCDATA        $0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0018 00024 (main.go:3)  FUNCDATA        $1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0018 00024 (main.go:3)  PCDATA  $1, $0
        0x0018 00024 (main.go:3)  CALL    runtime.makemap_small(SB)
        0x001c 00028 (main.go:3)  PCDATA  $0, $-2
        0x001c 00028 (main.go:3)  MOVWU   runtime.writeBarrier(SB), R1
        0x0024 00036 (main.go:3)  CBZW    R1, 60
        0x0028 00040 (main.go:3)  CALL    runtime.gcWriteBarrier2(SB)
        0x002c 00044 (main.go:3)  MOVD    R0, (R25)
        0x0030 00048 (main.go:3)  MOVD    main.a(SB), R1
        0x0038 00056 (main.go:3)  MOVD    R1, 8(R25)
        0x003c 00060 (main.go:3)  MOVD    R0, main.a(SB)
        0x0044 00068 (main.go:3)  LDP     -8(RSP), (R29, R30)
        0x0048 00072 (main.go:3)  ADD     $16, RSP
        0x004c 00076 (main.go:3)  RET     (R30)
        0x0050 00080 (main.go:3)  NOP
        0x0050 00080 (main.go:3)  PCDATA  $1, $-1
        0x0050 00080 (main.go:3)  PCDATA  $0, $-2
        0x0050 00080 (main.go:3)  MOVD    R30, R3
        0x0054 00084 (main.go:3)  CALL    runtime.morestack_noctxt(SB)
        0x0058 00088 (main.go:3)  PCDATA  $0, $-1
        0x0058 00088 (main.go:3)  JMP     0
        0x0000 90 0b 40 f9 ff 63 30 eb 49 02 00 54 fe 0f 1f f8  ..@..c0.I..T....
        0x0010 fd 83 1f f8 fd 23 00 d1 00 00 00 94 1b 00 00 90  .....#..........
        0x0020 61 03 40 b9 c1 00 00 34 00 00 00 94 20 03 00 f9  a.@....4.... ...
        0x0030 1b 00 00 90 61 03 40 f9 21 07 00 f9 1b 00 00 90  ....a.@.!.......
        0x0040 60 03 00 f9 fd fb 7f a9 ff 43 00 91 c0 03 5f d6  `........C...._.
        0x0050 e3 03 1e aa 00 00 00 94 ea ff ff 17 00 00 00 00  ................
        rel 24+4 t=9 runtime.makemap_small+0
        rel 28+8 t=40 runtime.writeBarrier+0
        rel 40+4 t=9 runtime.gcWriteBarrier2+0
        rel 48+8 t=41 main.a+0
        rel 60+8 t=41 main.a+0
        rel 84+4 t=9 runtime.morestack_noctxt+0
go:cuinfo.producer.main SDWARFCUINFO dupok size=0
        0x0000 2d 73 68 61 72 65 64 20 72 65 67 61 62 69        -shared regabi
go:cuinfo.packagename.main SDWARFCUINFO dupok size=0
        0x0000 6d 61 69 6e                                      main
main..inittask SNOPTRDATA size=16
        0x0000 00 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00  ................
        rel 8+8 t=1 main.init+0
go:string."key1" SRODATA dupok size=4
        0x0000 6b 65 79 31                                      key1
main.a SBSS size=8
 SDWARFVAR size=23
        0x0000 0a 6d 61 69 6e 2e 61 00 09 03 00 00 00 00 00 00  .main.a.........
        0x0010 00 00 00 00 00 00 01                             .......
        rel 10+8 t=1 main.a+0
        rel 18+4 t=31 go:info.map[string]string+0
runtime.memequal64·f SRODATA dupok size=8
        0x0000 00 00 00 00 00 00 00 00                          ........
        rel 0+8 t=1 runtime.memequal64+0
runtime.gcbits.0100000000000000 SRODATA dupok size=8
        0x0000 01 00 00 00 00 00 00 00                          ........
type:.namedata.*[8]uint8- SRODATA dupok size=11
        0x0000 00 09 2a 5b 38 5d 75 69 6e 74 38                 ..*[8]uint8
type:*[8]uint8 SRODATA dupok size=56
        0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
        0x0010 f8 9a 95 1a 08 08 08 36 00 00 00 00 00 00 00 00  .......6........
        0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0030 00 00 00 00 00 00 00 00                          ........
        rel 24+8 t=1 runtime.memequal64·f+0
        rel 32+8 t=1 runtime.gcbits.0100000000000000+0
        rel 40+4 t=5 type:.namedata.*[8]uint8-+0
        rel 48+8 t=1 type:[8]uint8+0
runtime.gcbits. SRODATA dupok size=0
type:[8]uint8 SRODATA dupok size=72
        0x0000 08 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0010 89 18 9c b4 0a 01 01 11 00 00 00 00 00 00 00 00  ................
        0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0040 08 00 00 00 00 00 00 00                          ........
        rel 24+8 t=1 runtime.memequal64·f+0
        rel 32+8 t=1 runtime.gcbits.+0
        rel 40+4 t=5 type:.namedata.*[8]uint8-+0
        rel 44+4 t=-32763 type:*[8]uint8+0
        rel 48+8 t=1 type:uint8+0
        rel 56+8 t=1 type:[]uint8+0
type:.namedata.*[8]string- SRODATA dupok size=12
        0x0000 00 0a 2a 5b 38 5d 73 74 72 69 6e 67              ..*[8]string
type:*[8]string SRODATA dupok size=56
        0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
        0x0010 e3 bf d7 63 08 08 08 36 00 00 00 00 00 00 00 00  ...c...6........
        0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0030 00 00 00 00 00 00 00 00                          ........
        rel 24+8 t=1 runtime.memequal64·f+0
        rel 32+8 t=1 runtime.gcbits.0100000000000000+0
        rel 40+4 t=5 type:.namedata.*[8]string-+0
        rel 48+8 t=1 type:noalg.[8]string+0
runtime.gcbits.5555000000000000 SRODATA dupok size=8
        0x0000 55 55 00 00 00 00 00 00                          UU......
type:noalg.[8]string SRODATA dupok size=72
        0x0000 80 00 00 00 00 00 00 00 78 00 00 00 00 00 00 00  ........x.......
        0x0010 0c 1c ff 04 02 08 08 11 00 00 00 00 00 00 00 00  ................
        0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0040 08 00 00 00 00 00 00 00                          ........
        rel 32+8 t=1 runtime.gcbits.5555000000000000+0
        rel 40+4 t=5 type:.namedata.*[8]string-+0
        rel 44+4 t=-32763 type:*[8]string+0
        rel 48+8 t=1 type:string+0
        rel 56+8 t=1 type:[]string+0
type:.namedata.*map.bucket[string]string- SRODATA dupok size=27
        0x0000 00 19 2a 6d 61 70 2e 62 75 63 6b 65 74 5b 73 74  ..*map.bucket[st
        0x0010 72 69 6e 67 5d 73 74 72 69 6e 67                 ring]string
type:*map.bucket[string]string SRODATA dupok size=56
        0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
        0x0010 40 92 79 ff 08 08 08 36 00 00 00 00 00 00 00 00  @.y....6........
        0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0030 00 00 00 00 00 00 00 00                          ........
        rel 24+8 t=1 runtime.memequal64·f+0
        rel 32+8 t=1 runtime.gcbits.0100000000000000+0
        rel 40+4 t=5 type:.namedata.*map.bucket[string]string-+0
        rel 48+8 t=1 type:noalg.map.bucket[string]string+0
runtime.gcbits.aaaaaaaa02000000 SRODATA dupok size=8
        0x0000 aa aa aa aa 02 00 00 00                          ........
type:.importpath.. SRODATA dupok size=2
        0x0000 00 00                                            ..
type:.namedata.topbits- SRODATA dupok size=9
        0x0000 00 07 74 6f 70 62 69 74 73                       ..topbits
type:.namedata.keys- SRODATA dupok size=6
        0x0000 00 04 6b 65 79 73                                ..keys
type:.namedata.elems- SRODATA dupok size=7
        0x0000 00 05 65 6c 65 6d 73                             ..elems
type:.namedata.overflow- SRODATA dupok size=10
        0x0000 00 08 6f 76 65 72 66 6c 6f 77                    ..overflow
type:noalg.map.bucket[string]string SRODATA dupok size=176
        0x0000 10 01 00 00 00 00 00 00 10 01 00 00 00 00 00 00  ................
        0x0010 4d c0 63 4d 02 08 08 19 00 00 00 00 00 00 00 00  M.cM............
        0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0040 04 00 00 00 00 00 00 00 04 00 00 00 00 00 00 00  ................
        0x0050 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0060 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0070 00 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
        0x0080 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0090 88 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x00a0 00 00 00 00 00 00 00 00 08 01 00 00 00 00 00 00  ................
        rel 32+8 t=1 runtime.gcbits.aaaaaaaa02000000+0
        rel 40+4 t=5 type:.namedata.*map.bucket[string]string-+0
        rel 44+4 t=-32763 type:*map.bucket[string]string+0
        rel 48+8 t=1 type:.importpath..+0
        rel 56+8 t=1 type:noalg.map.bucket[string]string+80
        rel 80+8 t=1 type:.namedata.topbits-+0
        rel 88+8 t=1 type:[8]uint8+0
        rel 104+8 t=1 type:.namedata.keys-+0
        rel 112+8 t=1 type:noalg.[8]string+0
        rel 128+8 t=1 type:.namedata.elems-+0
        rel 136+8 t=1 type:noalg.[8]string+0
        rel 152+8 t=1 type:.namedata.overflow-+0
        rel 160+8 t=1 type:unsafe.Pointer+0
runtime.strhash·f SRODATA dupok size=8
        0x0000 00 00 00 00 00 00 00 00                          ........
        rel 0+8 t=1 runtime.strhash+0
type:.namedata.*map[string]string- SRODATA dupok size=20
        0x0000 00 12 2a 6d 61 70 5b 73 74 72 69 6e 67 5d 73 74  ..*map[string]st
        0x0010 72 69 6e 67                                      ring
type:*map[string]string SRODATA dupok size=56
        0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
        0x0010 d8 6c ad 45 08 08 08 36 00 00 00 00 00 00 00 00  .l.E...6........
        0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0030 00 00 00 00 00 00 00 00                          ........
        rel 24+8 t=1 runtime.memequal64·f+0
        rel 32+8 t=1 runtime.gcbits.0100000000000000+0
        rel 40+4 t=5 type:.namedata.*map[string]string-+0
        rel 48+8 t=1 type:map[string]string+0
type:map[string]string SRODATA dupok size=88
        0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
        0x0010 de 62 2b 92 02 08 08 35 00 00 00 00 00 00 00 00  .b+....5........
        0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0040 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0050 10 10 10 01 0c 00 00 00                          ........
        rel 32+8 t=1 runtime.gcbits.0100000000000000+0
        rel 40+4 t=5 type:.namedata.*map[string]string-+0
        rel 44+4 t=-32763 type:*map[string]string+0
        rel 48+8 t=1 type:string+0
        rel 56+8 t=1 type:string+0
        rel 64+8 t=1 type:noalg.map.bucket[string]string+0
        rel 72+8 t=1 runtime.strhash·f+0
main.main.func1·f SRODATA dupok size=8
        0x0000 00 00 00 00 00 00 00 00                          ........
        rel 0+8 t=1 main.main.func1+0
main.main.func2·f SRODATA dupok size=8
        0x0000 00 00 00 00 00 00 00 00                          ........
        rel 0+8 t=1 main.main.func2+0
gclocals·g2BeySu+wFnoycgXfElmcg== SRODATA dupok size=8
        0x0000 01 00 00 00 00 00 00 00                          ........
```

# 总结

# 引用

[走进Golang之编译器原理 | 小米信息部技术团队 (xiaomi-info.github.io)](https://xiaomi-info.github.io/2019/11/13/golang-compiler-principle/)

[走进Golang之运行与Plan9汇编 | 小米信息部技术团队 (xiaomi-info.github.io)](https://xiaomi-info.github.io/2019/11/27/golang-compiler-plan9/)

[golang-notes/assembly.md at master · cch123/golang-notes · GitHub](https://github.com/cch123/golang-notes/blob/master/assembly.md)

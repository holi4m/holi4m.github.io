---
layout: post
comments: true
title: "How to debugging a Kernel Driver"
tags:
- Windows
- Kernel
- Debugging
---

# [-] How to debugging a Kernel Driver

## 1. Introduce

해당 포스팅은 드라이버를 디버깅 할 때 접근할 수 있는 방법에 대한 포스팅입니다.

this posting describe how to debugging a kernel driver in a windbg.

### 1.1 Testbed

> Host: Windows 10 19041.1052 and VirtualKD
Guest: Windows 10 19041.985 and VirtualKD
Debugger: Windbg

## 2. Why we debugging a kernel driver?

대부분의 게임 핵과 안티치트 솔루션은 커널 드라이버를 사용합니다.
Most of a gamehack and anti-cheat solution are run with a kernel driver.

게임핵을 막고 안티 치트 솔루션을 연구 및 개발하려면 커널 드라이버에 대한 지식은 필수적입니다.
Knowledge of a kernel driver is necessary for prevent a gamehack and research anti-cheat solution.

## 3. How to debugging a Kenrel Driver

 해당 포스팅에서는 두가지의 커널 디버깅 방법을 설명합니다.
This posting describe a two way about kernel driver debugging in a windbg.

### 3.1 Set Exception Command

windbg의 Set Exception Command를 사용하여 특정 드라이버 로드 시점부터 디버깅이 가능합니다.
you can use the sxe command to debug when the driver is loaded

```c
// sxe ld "DriverName.sys"

0: kd> sxe ld debugging.sys
0: kd> g
```

해당 드라이버를 로드하면 Exception이 발생하며, DriverEntry에 Breakpoint를 Set 하고 디버깅을 시작합니다.
When the driver is loaded, an Exception occurs, and sets a breakpoint to the DriverEntry and start debugging.

```c
nt!DebugService2+0x5:
fffff802`44e13be5 cc              int     3
1: kd> lm
start             end                 module name
00007ffc`96650000 00007ffc`96682000   vertdll    (deferred)             
00007ffc`96690000 00007ffc`96885000   ntdll    T (no symbols)           
fffff802`44560000 fffff802`447ef000   mcupdate_GenuineIntel   (deferred)             
...           
**fffff802`4a1c0000 fffff802`4a1c7000   Debugging   (deferred)**             
...

1: kd> bp Debugging!DriverEntry
1: kd> g
Breakpoint 0 hit
Debugging!DriverEntry:
fffff802`4a1c1000 4053            push    rbx
```

### 3.2 PnpCallDriverEntry BreakPoint

드라이버 로드 루틴에서 호출되는 PnpCallDriverEntry 함수에 BreakPoint를 설정해 디버깅 하는 방법입니다.
and you can debug other way, sets a breakpoint to the PnpCallDriverEntry Function in a driverload routine

해당 방법은 Driver 심볼이 없거나, 패킹된 드라이버에 주로 사용합니다.
The method is mainly used when you don't have a driver symbol or you wanna debug a packed driver

```c
0: kd> bp pnpcalldriverentry
0: kd> bl
     0 e Disable Clear  fffff805`7f375f00     0001 (0001) nt!PnpCallDriverEntry

0: kd> g
Breakpoint 0 hit
nt!PnpCallDriverEntry:
fffff805`7f375f00 4c8bdc          mov     r11,rsp

1: kd> kn
 # Child-SP          RetAddr           Call Site
00 ffff8185`31ca0958 fffff805`7f3421f5 nt!PnpCallDriverEntry
01 ffff8185`31ca0960 fffff805`7f386cc7 nt!IopLoadDriver+0x4e5
02 ffff8185`31ca0b30 fffff805`7ef5a225 nt!IopLoadUnloadDriver+0x57
03 ffff8185`31ca0b70 fffff805`7ef0e3b5 nt!ExpWorkerThread+0x105
04 ffff8185`31ca0c10 fffff805`7f017348 nt!PspSystemThreadStartup+0x55
05 ffff8185`31ca0c60 00000000`00000000 nt!KiStartSystemThread+0x28

0: kd> u rip l20
nt!PnpCallDriverEntry:
fffff805`7f375f00 4c8bdc          mov     r11,rsp
fffff805`7f375f03 49895b08        mov     qword ptr [r11+8],rbx
fffff805`7f375f07 49897310        mov     qword ptr [r11+10h],rsi
fffff805`7f375f0b 57              push    rdi
fffff805`7f375f0c 4883ec50        sub     rsp,50h
fffff805`7f375f10 498363d800      and     qword ptr [r11-28h],0
fffff805`7f375f15 488bda          mov     rbx,rdx
fffff805`7f375f18 49894be0        mov     qword ptr [r11-20h],rcx
fffff805`7f375f1c 498d53d8        lea     rdx,[r11-28h]
fffff805`7f375f20 65488b042588010000 mov   rax,qword ptr gs:[188h]
fffff805`7f375f29 488bf9          mov     rdi,rcx
fffff805`7f375f2c b905000000      mov     ecx,5
fffff805`7f375f31 498943e8        mov     qword ptr [r11-18h],rax
fffff805`7f375f35 e8b6c5eeff      call    nt!PnpEnableWatchdog (fffff805`7f2624f0)
fffff805`7f375f3a 488bf0          mov     rsi,rax
fffff805`7f375f3d 488bd3          mov     rdx,rbx
fffff805`7f375f40 488b4758        mov     rax,qword ptr [rdi+58h]
fffff805`7f375f44 488bcf          mov     rcx,rdi
**fffff805`7f375f47 e84429caff      call    nt!guard_dispatch_icall (fffff805`7f018890)**
fffff805`7f375f4c 8bf8            mov     edi,eax
fffff805`7f375f4e 4885f6          test    rsi,rsi
fffff805`7f375f51 7454            je      nt!PnpCallDriverEntry+0xa7 (fffff805`7f375fa7)
fffff805`7f375f53 488b5e08        mov     rbx,qword ptr [rsi+8]
fffff805`7f375f57 41b001          mov     r8b,1
fffff805`7f375f5a 4533c9          xor     r9d,r9d
fffff805`7f375f5d 418ad0          mov     dl,r8b
fffff805`7f375f60 488b4b38        mov     rcx,qword ptr [rbx+38h]
fffff805`7f375f64 e8d783b1ff      call    nt!ExDeleteTimer (fffff805`7ee8e340)
fffff805`7f375f69 4883633800      and     qword ptr [rbx+38h],0
fffff805`7f375f6e 8b4360          mov     eax,dword ptr [rbx+60h]
fffff805`7f375f71 85c0            test    eax,eax
fffff805`7f375f73 0f8f0b1b0d00    jg      nt!PnpCallDriverEntry+0xd1b84 (fffff805`7f447a84)
```

해당 함수 내의 CFG 함수에서 DriverEntry를 체크하고 호출합니다.
Checked and called the driverentry in the CFG function within PnpCallDriverEntry
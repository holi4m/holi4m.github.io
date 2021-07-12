---
layout: post
comments: true
title: "Memory Descriptor List"
tags:
- Windows
- MDL
---
# [-] Memory Descriptor List

## 1. Introduce

커널 인라인 후킹을 하기 위하여 WriteProtect( CR0 레지스터 내 WP)를 비활성화 하기 위해서 입니다.

## 2. Memory Descriptor List (MDL)

가상 메모리(Virtual Address)는 연속된 메모리 주소를 가지고 있으나. 물리 메모리(Physical Memory)의 경우 연속적이지 않을 수 있으며

OS는 MDL을 이용하여 가상 메모리(Virtual Memory)의 물리 메모리(Physical Memory) 페이지 레이아웃을 설명한다.

라고 MSDN에서 설명하며

또한 MDL은 반투명한(semi-opque) 구조체라 Next, MdlFlags를 제외한 다른 멤버에 대하여 직접 접근을 하지 말 것을 권고하며, 접근을 할 경우에 특정 매크로를 사용하라고 설명하고 있습니다.

Mdl의 구조체와 해당 맴버의 정보를 얻는 매크로들을 주석으로 작성하였습니다.

```c
0: kd> dt _MDL
nt!_MDL
   +0x000 Next             : Ptr64 _MDL // Direct Access
   +0x008 Size             : Int2B // MmGetMdlByteCount
   +0x00a MdlFlags         : Int2B // Direct Access
   +0x00c AllocationProcessorNumber : Uint2B
   +0x00e Reserved         : Uint2B
   +0x010 Process          : Ptr64 _EPROCESS
   +0x018 MappedSystemVa   : Ptr64 Void // MmGetSystemAddressForMdlSafe
   +0x020 StartVa          : Ptr64 Void
   +0x028 ByteCount        : Uint4B // MmGetMdlByteCount
   +0x02c ByteOffset       : Uint4B // MmGetMdlByteOffset

// +0x30 physical address area // non offical

// #define MmGetMdlVirtualAddress(Mdl)    
// (PVOID) ((PCHAR) ((Mdl)->StartVa) + (Mdl)->ByteOffset))
```

## 3. PoC

PoC에 사용할 소스입니다.

```c
/*
#Name : driver.c
#Desc : Driver Main
*/

#include "define.h"

NTSTATUS DriverEntry(IN PDRIVER_OBJECT pDriver, IN PUNICODE_STRING pRegPath)
{
	UNREFERENCED_PARAMETER(pDriver);
	UNREFERENCED_PARAMETER(pRegPath);
	pDriver->DriverUnload = UnloadDriver;

	DbgPrintEx(DPFLTR_ACPI_ID, 0, "[+] Load Driver\n");
	PMDL pMDL = NULL;
	pMDL = IoAllocateMdl(ExAllocatePoolWithTag, 0x10, FALSE, FALSE, NULL);
	/*
	PMDL IoAllocateMdl(
	  __drv_aliasesMem PVOID VirtualAddress, // [in, optional] MDL이 설명할 가상 주소 포인터
	  ULONG                  Length, // [in] MDL이 설명할 길이
	  BOOLEAN                SecondaryBuffer,  // [in] 버퍼가 기본버퍼인지 보조 버퍼인지 결정 여부를 나타내는 변수.
	  BOOLEAN                ChargeQuota, // Reserved 변수. 무조건 FALSE
	  PIRP                   Irp // MDL과 연관 될 IRP에 대한 포인터. 
	);
	*/
	__try
	{
		MmProbeAndLockPages(pMDL, KernelMode, IoReadAccess);
		/*
		void MmProbeAndLockPages(
		  PMDL            MemoryDescriptorList, // MDL 포인터
		  KPROCESSOR_MODE AccessMode, // AccessMode에 대한 값 "KernalMode" 또는 "UserMode" 중 선택
		  LOCK_OPERATION  Operation // 
		);
		*/
	}
	__except(EXCEPTION_EXECUTE_HANDLER)
	{
		IoFreeMdl(pMDL);
		DbgPrintEx(DPFLTR_ACPI_ID, 0, "[!] IoAllocateMdl Error\n");
		return STATUS_FAILED_DRIVER_ENTRY;
	}

	PVOID MappingData = MmMapLockedPagesSpecifyCache(pMDL, KernelMode, MmNonCached, NULL, FALSE, NormalPagePriority);
	if (MappingData)
	{
		MmUnmapLockedPages(MappingData, pMDL);
	}
	MmUnlockPages(pMDL);
	IoFreeMdl(pMDL);

	return STATUS_SUCCESS;

}

VOID UnloadDriver(IN PDRIVER_OBJECT pDriver)
{
	UNREFERENCED_PARAMETER(pDriver);
	
	DbgPrintEx(DPFLTR_ACPI_ID, 0, "[+] Unload Driver\n");
}
```

```c
/*
# Name : define.h
# Desc : 사용할 함수를 정의함
*/
#pragma once
#include <ntifs.h>

#pragma warning(disable: 4152) // nonstandar extension warning disable

// define DrvierEntry, Unload Driver
NTSTATUS DriverEntry(IN PDRIVER_OBJECT pDriver, IN PUNICODE_STRING pRegPath);
VOID UnloadDriver(IN PDRIVER_OBJECT pDriver);
```

### 3.0 MdlFlags

wdm.h 에 정의되어 있는 MdlFlags입니다.

```c
#define MDL_MAPPED_TO_SYSTEM_VA     0x0001
#define MDL_PAGES_LOCKED            0x0002
#define MDL_SOURCE_IS_NONPAGED_POOL 0x0004
#define MDL_ALLOCATED_FIXED_SIZE    0x0008
#define MDL_PARTIAL                 0x0010
#define MDL_PARTIAL_HAS_BEEN_MAPPED 0x0020
#define MDL_IO_PAGE_READ            0x0040
#define MDL_WRITE_OPERATION         0x0080
#define MDL_LOCKED_PAGE_TABLES      0x0100
#define MDL_PARENT_MAPPED_SYSTEM_VA MDL_LOCKED_PAGE_TABLES
#define MDL_FREE_EXTRA_PTES         0x0200
#define MDL_DESCRIBES_AWE           0x0400
#define MDL_IO_SPACE                0x0800
#define MDL_NETWORK_HEADER          0x1000
#define MDL_MAPPING_CAN_FAIL        0x2000
#define MDL_PAGE_CONTENTS_INVARIANT 0x4000
#define MDL_ALLOCATED_MUST_SUCCEED  MDL_PAGE_CONTENTS_INVARIANT
#define MDL_INTERNAL                0x8000

#define MDL_MAPPING_FLAGS (MDL_MAPPED_TO_SYSTEM_VA     | \
                           MDL_PAGES_LOCKED            | \
                           MDL_SOURCE_IS_NONPAGED_POOL | \
                           MDL_PARTIAL_HAS_BEEN_MAPPED | \
                           MDL_PARENT_MAPPED_SYSTEM_VA | \
                           MDL_SYSTEM_VA               | \
                           MDL_IO_SPACE )
```

### 3.1 IoAllocateMdl

IoAllocateMdl을 호출한 뒤 MDL입니다.

```c
1: kd> dt _mdl ffffc60c83a9fc20
nt!_MDL
   +0x000 Next             : (null) 
   +0x008 Size             : 0n56
   +0x00a MdlFlags         : 0n8 // MDL_ALLOCATED_FIXED_SIZE
   +0x00c AllocationProcessorNumber : 1
   +0x00e Reserved         : 0xffff
   +0x010 Process          : 0xffffc60c`81bb9340 _EPROCESS
   +0x018 MappedSystemVa   : 0xffff8981`99ce5928 Void
   +0x020 StartVa          : 0xfffff807`46dc9000 Void
   +0x028 ByteCount        : 0x10
   +0x02c ByteOffset       : 0x160

//physical address area
1: kd> dp ffffc60c83a9fc20+0x30
ffffc60c`83a9fc50  00000000`00023c7e 00000000`0007d68f
ffffc60c`83a9fc60  00000000`00004e08 00000000`00005109
ffffc60c`83a9fc70  006d0061`0069006c 00700070`0041005c
ffffc60c`83a9fc80  00610074`00610044 0063006f`004c005c
ffffc60c`83a9fc90  004d005c`006c0061 006f0072`00630069
ffffc60c`83a9fca0  00740066`006f0073 0065006e`004f005c
ffffc60c`83a9fcb0  00760069`00720044 006e004f`005c0065
ffffc60c`83a9fcc0  00690072`00440065 0065002e`00650076

1: kd> !db 23c7e*1000+0x160 // (MDL+0x30)*1000 + ByteOffset 
#23c7e160 5c 00 77 00 69 00 6e 00-65 00 76 00 74 00 5c 00 \.w.i.n.e.v.t.\.
#23c7e170 4c 00 6f 00 67 00 73 00-5c 00 53 00 79 00 73 00 L.o.g.s.\.S.y.s.
#23c7e180 74 00 65 00 6d 00 2e 00-65 00 76 00 74 00 78 00 t.e.m...e.v.t.x.
#23c7e190 00 00 78 00 00 00 38 00-45 00 34 00 45 00 37 00 ..x...8.E.4.E.7.
#23c7e1a0 43 00 2e 00 70 00 66 00-00 00 35 00 43 00 2d 00 C...p.f...5.C.-.
#23c7e1b0 41 00 42 00 35 00 45 00-2d 00 34 00 44 00 39 00 A.B.5.E.-.4.D.9.
#23c7e1c0 41 00 2d 00 42 00 42 00-41 00 31 00 2d 00 37 00 A.-.B.B.A.1.-.7.
#23c7e1d0 34 00 35 00 39 00 41 00-41 00 35 00 34 00 32 00 4.5.9.A.A.5.4.2.
```

IoAllocateMdl 호출한 뒤 아직 Physical Address에는 ExAllocatePoolWithTag가 아직 할당되어 있지 않습니다.

### 3.2 MmProbeAndLockPages

MmProbeAndLockPages을 호출한 뒤 MDL입니다. 

```c
1: kd> dt _mdl ffffc60c83a9fc20
nt!_MDL
   +0x000 Next             : (null) 
   +0x008 Size             : 0n56
   **+0x00a MdlFlags         : 0n10** // MDL_ALLOCATED_FIXED_SIZE | MDL_PAGES_LOCKED 
   +0x00c AllocationProcessorNumber : 1
   +0x00e Reserved         : 0xffff
   +0x010 Process          : (null) 
   +0x018 MappedSystemVa   : 0xffff8981`99ce5928 Void
   +0x020 StartVa          : 0xfffff807`46dc9000 Void
   +0x028 ByteCount        : 0x10
   +0x02c ByteOffset       : 0x160

1: kd> dp ffffc60c83a9fc20+0x30
ffffc60c`83a9fc50  **00000000`00003db3** 00000000`0007d68f
ffffc60c`83a9fc60  00000000`00004e08 00000000`00005109
ffffc60c`83a9fc70  006d0061`0069006c 00700070`0041005c
ffffc60c`83a9fc80  00610074`00610044 0063006f`004c005c
ffffc60c`83a9fc90  004d005c`006c0061 006f0072`00630069
ffffc60c`83a9fca0  00740066`006f0073 0065006e`004f005c
ffffc60c`83a9fcb0  00760069`00720044 006e004f`005c0065
ffffc60c`83a9fcc0  00690072`00440065 0065002e`00650076

1: kd> !db 3db3*1000 +0x160 //(MDL+0x30)*1000 + ByteOffset 
# 3db3160 48 89 5c 24 08 48 89 6c-24 10 48 89 74 24 18 57 H.\$.H.l$.H.t$.W
# 3db3170 41 56 41 57 48 83 ec 30-65 48 8b 04 25 20 00 00 AVAWH..0eH..% ..
# 3db3180 00 45 8b f0 44 0f b7 3d-74 6e 34 00 48 8b ea 8b .E..D..=tn4.H...
# 3db3190 f1 4c 8b 88 c0 00 00 00-41 0f b7 b9 92 00 00 00 .L......A.......
# 3db31a0 0f ba ef 1f 0f ba f7 1f-33 db 8b c7 89 5c 24 68 ........3....\$h
# 3db31b0 44 8b c8 89 5c 24 20 45-8b c6 48 8b d5 8b ce e8 D...\$ E..H.....
# 3db31c0 cc 23 96 ff 48 85 c0 0f-84 2b 00 00 00 48 8b d8 .#..H....+...H..
# 3db31d0 48 8b 6c 24 58 48 8b c3-48 8b 5c 24 50 48 8b 74 H.l$XH..H.\$PH.t

1: kd> db ExAllocatePoolWithTag
fffff807`46dc9160  48 89 5c 24 08 48 89 6c-24 10 48 89 74 24 18 57  H.\$.H.l$.H.t$.W
fffff807`46dc9170  41 56 41 57 48 83 ec 30-65 48 8b 04 25 20 00 00  AVAWH..0eH..% ..
fffff807`46dc9180  00 45 8b f0 44 0f b7 3d-74 6e 34 00 48 8b ea 8b  .E..D..=tn4.H...
fffff807`46dc9190  f1 4c 8b 88 c0 00 00 00-41 0f b7 b9 92 00 00 00  .L......A.......
fffff807`46dc91a0  0f ba ef 1f 0f ba f7 1f-33 db 8b c7 89 5c 24 68  ........3....\$h
fffff807`46dc91b0  44 8b c8 89 5c 24 20 45-8b c6 48 8b d5 8b ce e8  D...\$ E..H.....
fffff807`46dc91c0  cc 23 96 ff 48 85 c0 0f-84 2b 00 00 00 48 8b d8  .#..H....+...H..
fffff807`46dc91d0  48 8b 6c 24 58 48 8b c3-48 8b 5c 24 50 48 8b 74  H.l$XH..H.\$PH.t
```

MdlFlags에 MDL_PAGES_LOCKED가 추가되었고 Physical Memory 주소값이 변경되었습니다.

또한 바뀐 Physical Address 주소에 ExAllocatePoolWithTag 함수가 매핑되었습니다.

### 3.3 MmMapLockedPagesSpecifyCache

MmMapLockedPagesSpecifyCache을 호출한 뒤 MDL입니다.

```c
1: kd> dt _mdl ffffc60c83a9fc20
nt!_MDL
   +0x000 Next             : (null) 
   +0x008 Size             : 0n56
   **+0x00a MdlFlags         : 0n11** // MDL_ALLOCATED_FIXED_SIZE | MDL_PAGES_LOCKED | MDL_MAPPED_TO_SYSTEM_VA  
   +0x00c AllocationProcessorNumber : 1
   +0x00e Reserved         : 0xffff
   +0x010 Process          : (null) 
   **+0x018 MappedSystemVa   : 0xffff8981`99ce6160 Void // Mapped**
   +0x020 StartVa          : 0xfffff807`46dc9000 Void
   +0x028 ByteCount        : 0x10
   +0x02c ByteOffset       : 0x160

1: kd> dp ffffc60c83a9fc20+0x30
ffffc60c`83a9fc50  00000000`00003db3 00000000`0007d68f
ffffc60c`83a9fc60  00000000`00004e08 00000000`00005109
ffffc60c`83a9fc70  006d0061`0069006c 00700070`0041005c
ffffc60c`83a9fc80  00610074`00610044 0063006f`004c005c
ffffc60c`83a9fc90  004d005c`006c0061 006f0072`00630069
ffffc60c`83a9fca0  00740066`006f0073 0065006e`004f005c
ffffc60c`83a9fcb0  00760069`00720044 006e004f`005c0065
ffffc60c`83a9fcc0  00690072`00440065 0065002e`00650076

1: kd> !db 3db3*1000 +0x160
# 3db3160 48 89 5c 24 08 48 89 6c-24 10 48 89 74 24 18 57 H.\$.H.l$.H.t$.W
# 3db3170 41 56 41 57 48 83 ec 30-65 48 8b 04 25 20 00 00 AVAWH..0eH..% ..
# 3db3180 00 45 8b f0 44 0f b7 3d-74 6e 34 00 48 8b ea 8b .E..D..=tn4.H...
# 3db3190 f1 4c 8b 88 c0 00 00 00-41 0f b7 b9 92 00 00 00 .L......A.......
# 3db31a0 0f ba ef 1f 0f ba f7 1f-33 db 8b c7 89 5c 24 68 ........3....\$h
# 3db31b0 44 8b c8 89 5c 24 20 45-8b c6 48 8b d5 8b ce e8 D...\$ E..H.....
# 3db31c0 cc 23 96 ff 48 85 c0 0f-84 2b 00 00 00 48 8b d8 .#..H....+...H..
# 3db31d0 48 8b 6c 24 58 48 8b c3-48 8b 5c 24 50 48 8b 74 H.l$XH..H.\$PH.t

1: kd> db 0xffff8981`99ce6160 // MappedSystemVa
ffff8981`99ce6160  48 89 5c 24 08 48 89 6c-24 10 48 89 74 24 18 57  H.\$.H.l$.H.t$.W
ffff8981`99ce6170  41 56 41 57 48 83 ec 30-65 48 8b 04 25 20 00 00  AVAWH..0eH..% ..
ffff8981`99ce6180  00 45 8b f0 44 0f b7 3d-74 6e 34 00 48 8b ea 8b  .E..D..=tn4.H...
ffff8981`99ce6190  f1 4c 8b 88 c0 00 00 00-41 0f b7 b9 92 00 00 00  .L......A.......
ffff8981`99ce61a0  0f ba ef 1f 0f ba f7 1f-33 db 8b c7 89 5c 24 68  ........3....\$h
ffff8981`99ce61b0  44 8b c8 89 5c 24 20 45-8b c6 48 8b d5 8b ce e8  D...\$ E..H.....
ffff8981`99ce61c0  cc 23 96 ff 48 85 c0 0f-84 2b 00 00 00 48 8b d8  .#..H....+...H..
ffff8981`99ce61d0  48 8b 6c 24 58 48 8b c3-48 8b 5c 24 50 48 8b 74  H.l$XH..H.\$PH.t
```

MappedSystemVa 주소값이 변경되었고 ExAllocatePoolWithTag가 Mapped 되었습니다.

해당 방법을 통하여 Kernel Inline 후킹이 가능합니다.

1. 후킹 할 함수를 MDL로 매핑 합니다.
2. [MmProtectMdlSystemAddress](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-mmprotectmdlsystemaddress) 함수를 이용하여 매핑 한 MDL의 Protect를 PAGE_READWRITE로 변경합니다.
3. 인라인 패치를 수행합니다.

## 4. Reference

1. [https://shhoya.github.io/windows_MDL.html#0x01-memory-descriptor-list](https://shhoya.github.io/windows_MDL.html#0x01-memory-descriptor-list)
2. [https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/using-mdls](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/using-mdls)
3. [http://bsodtutorials.blogspot.com/2013/12/understanding-mdls-memory-descriptor.html](http://bsodtutorials.blogspot.com/2013/12/understanding-mdls-memory-descriptor.html)
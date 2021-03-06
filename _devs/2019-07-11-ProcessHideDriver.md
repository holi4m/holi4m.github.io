---
layout: post
title: "Process Hide Driver (DKOM)"
tags:
- Windows
- Development
- Kernel
---
# # Introduce
기본적인 DKOM인 Process Hide Driver를 개발 해 보았습니다.

# # Testbed
> Windows 10 1809 build 17763.253

### [#]Driver.c
notepad.exe의 Process를 숨기는 드라이버 소스코드입니다.
DKOM(Direct Kernel Object Manipulation)을 할 때 기본적으로 변조하는 ActiveProcessLinks를 변조하였습니다.

```cpp
#include <ntifs.h>

VOID Unload(_In_ PDRIVER_OBJECT pDriverObject)
{
	UNREFERENCED_PARAMETER(pDriverObject);
	DbgPrint("Driver Unload\n");
	return;
}

NTSTATUS DriverEntry(
	_In_ PDRIVER_OBJECT pDriverObject,
	_In_ PUNICODE_STRING pRegistryPath
)
{
	UNREFERENCED_PARAMETER(pDriverObject);
	UNREFERENCED_PARAMETER(pRegistryPath);

	DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[Driver Load]\n");
	PEPROCESS eped = NULL;
	eped = (PEPROCESS)PsGetCurrentProcess();
	PLIST_ENTRY pNode, pBuf;
	pNode = NULL;
	pBuf = NULL;
	unsigned char* peped = NULL;
	peped = (unsigned char*)eped;
	pNode = pBuf = (PLIST_ENTRY)(peped + 0x2e8);

	while (1)
	{
		if (strncmp("notepad.exe", (const char*)((unsigned char*)pNode + 0x168), 14) == 0)
		{
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[Process Name is (%s)]\nBefore\n",(const char*)((unsigned char*)pNode + 0x160));
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[pBuf->Flink = %p]\n", (const char*)((unsigned char*)pBuf->Flink));
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[pNode->Blink = %p]\n", (const char*)((unsigned char*)pNode->Flink));
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[pNode->Flink->Blink = %p]\n", (const char*)((unsigned char*)pNode->Flink->Blink));
			pBuf->Flink = pNode->Flink;
			pNode->Flink->Blink = pNode->Blink;
			pNode->Flink = pNode;
			pNode->Blink = pNode;
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "After \n[pBuf->Flink = %p]\n", (const char*)((unsigned char*)pBuf->Flink));
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[pNode->Flink = %p]\n", (const char*)((unsigned char*)pNode->Flink));
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[pNode->Flink->Blink = %p]\n", (const char*)((unsigned char*)pNode->Flink->Blink));
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[pNode->Flink = %p]\n", (const char*)((unsigned char*)pNode->Blink));
			break;
		}
		pBuf = pNode;
		pNode = pNode->Flink;
	}
	
	pDriverObject->DriverUnload = Unload;
	return STATUS_SUCCESS;
}
```

1. PsGetCurrentProcess를 이용하여 _EPROCESS 구조체의 주소를 획득합니다.
2. 빌드에 맞게 pNode와 ImageFileName offset을 수정합니다.

### [#]_EPROCESS 
_eprocess 구조체입니다.

```
0: kd> dt NT!_eprocess
   +0x000 Pcb              : _KPROCESS
   +0x2e0 ProcessLock      : _EX_PUSH_LOCK
   +0x2e8 UniqueProcessId  : Ptr64 Void // PID
   +0x2f0 ActiveProcessLinks : _LIST_ENTRY // LINKED LIST
   +0x300 RundownProtect   : _EX_RUNDOWN_REF
   +0x308 Flags2           : Uint4B
   +0x308 JobNotReallyActive : Pos 0, 1 Bit
   +0x308 AccountingFolded : Pos 1, 1 Bit
   +0x308 NewProcessReported : Pos 2, 1 Bit
   +0x308 ExitProcessReported : Pos 3, 1 Bit
   +0x308 ReportCommitChanges : Pos 4, 1 Bit
   +0x308 LastReportMemory : Pos 5, 1 Bit
   +0x308 ForceWakeCharge  : Pos 6, 1 Bit
   +0x308 CrossSessionCreate : Pos 7, 1 Bit
   +0x308 NeedsHandleRundown : Pos 8, 1 Bit
   +0x308 RefTraceEnabled  : Pos 9, 1 Bit
   +0x308 PicoCreated      : Pos 10, 1 Bit
   +0x308 EmptyJobEvaluated : Pos 11, 1 Bit
   +0x308 DefaultPagePriority : Pos 12, 3 Bits
   +0x308 PrimaryTokenFrozen : Pos 15, 1 Bit
   +0x308 ProcessVerifierTarget : Pos 16, 1 Bit
   +0x308 RestrictSetThreadContext : Pos 17, 1 Bit
   +0x308 AffinityPermanent : Pos 18, 1 Bit
   +0x308 AffinityUpdateEnable : Pos 19, 1 Bit
   +0x308 PropagateNode    : Pos 20, 1 Bit
   +0x308 ExplicitAffinity : Pos 21, 1 Bit
   +0x308 ProcessExecutionState : Pos 22, 2 Bits
   +0x308 EnableReadVmLogging : Pos 24, 1 Bit
   +0x308 EnableWriteVmLogging : Pos 25, 1 Bit
   +0x308 FatalAccessTerminationRequested : Pos 26, 1 Bit
   +0x308 DisableSystemAllowedCpuSet : Pos 27, 1 Bit
   +0x308 ProcessStateChangeRequest : Pos 28, 2 Bits
   +0x308 ProcessStateChangeInProgress : Pos 30, 1 Bit
   +0x308 InPrivate        : Pos 31, 1 Bit
   +0x30c Flags            : Uint4B
   +0x30c CreateReported   : Pos 0, 1 Bit
   +0x30c NoDebugInherit   : Pos 1, 1 Bit
   +0x30c ProcessExiting   : Pos 2, 1 Bit
   +0x30c ProcessDelete    : Pos 3, 1 Bit
   +0x30c ManageExecutableMemoryWrites : Pos 4, 1 Bit
   +0x30c VmDeleted        : Pos 5, 1 Bit
   +0x30c OutswapEnabled   : Pos 6, 1 Bit
   +0x30c Outswapped       : Pos 7, 1 Bit
   +0x30c FailFastOnCommitFail : Pos 8, 1 Bit
   +0x30c Wow64VaSpace4Gb  : Pos 9, 1 Bit
   +0x30c AddressSpaceInitialized : Pos 10, 2 Bits
   +0x30c SetTimerResolution : Pos 12, 1 Bit
   +0x30c BreakOnTermination : Pos 13, 1 Bit
   +0x30c DeprioritizeViews : Pos 14, 1 Bit
   +0x30c WriteWatch       : Pos 15, 1 Bit
   +0x30c ProcessInSession : Pos 16, 1 Bit
   +0x30c OverrideAddressSpace : Pos 17, 1 Bit
   +0x30c HasAddressSpace  : Pos 18, 1 Bit
   +0x30c LaunchPrefetched : Pos 19, 1 Bit
   +0x30c Background       : Pos 20, 1 Bit
   +0x30c VmTopDown        : Pos 21, 1 Bit
   +0x30c ImageNotifyDone  : Pos 22, 1 Bit
   +0x30c PdeUpdateNeeded  : Pos 23, 1 Bit
   +0x30c VdmAllowed       : Pos 24, 1 Bit
   +0x30c ProcessRundown   : Pos 25, 1 Bit
   +0x30c ProcessInserted  : Pos 26, 1 Bit
   +0x30c DefaultIoPriority : Pos 27, 3 Bits
   +0x30c ProcessSelfDelete : Pos 30, 1 Bit
   +0x30c SetTimerResolutionLink : Pos 31, 1 Bit
   +0x310 CreateTime       : _LARGE_INTEGER
   +0x318 ProcessQuotaUsage : [2] Uint8B
   +0x328 ProcessQuotaPeak : [2] Uint8B
   +0x338 PeakVirtualSize  : Uint8B
   +0x340 VirtualSize      : Uint8B
   +0x348 SessionProcessLinks : _LIST_ENTRY
   +0x358 ExceptionPortData : Ptr64 Void
   +0x358 ExceptionPortValue : Uint8B
   +0x358 ExceptionPortState : Pos 0, 3 Bits
   +0x360 Token            : _EX_FAST_REF
   +0x368 MmReserved       : Uint8B
   +0x370 AddressCreationLock : _EX_PUSH_LOCK
   +0x378 PageTableCommitmentLock : _EX_PUSH_LOCK
   +0x380 RotateInProgress : Ptr64 _ETHREAD
   +0x388 ForkInProgress   : Ptr64 _ETHREAD
   +0x390 CommitChargeJob  : Ptr64 _EJOB
   +0x398 CloneRoot        : _RTL_AVL_TREE
   +0x3a0 NumberOfPrivatePages : Uint8B
   +0x3a8 NumberOfLockedPages : Uint8B
   +0x3b0 Win32Process     : Ptr64 Void
   +0x3b8 Job              : Ptr64 _EJOB
   +0x3c0 SectionObject    : Ptr64 Void
   +0x3c8 SectionBaseAddress : Ptr64 Void
   +0x3d0 Cookie           : Uint4B
   +0x3d8 WorkingSetWatch  : Ptr64 _PAGEFAULT_HISTORY
   +0x3e0 Win32WindowStation : Ptr64 Void
   +0x3e8 InheritedFromUniqueProcessId : Ptr64 Void
   +0x3f0 OwnerProcessId   : Uint8B
   +0x3f8 Peb              : Ptr64 _PEB
   +0x400 Session          : Ptr64 _MM_SESSION_SPACE
   +0x408 Spare1           : Ptr64 Void
   +0x410 QuotaBlock       : Ptr64 _EPROCESS_QUOTA_BLOCK
   +0x418 ObjectTable      : Ptr64 _HANDLE_TABLE
   +0x420 DebugPort        : Ptr64 Void
   +0x428 WoW64Process     : Ptr64 _EWOW64PROCESS
   +0x430 DeviceMap        : Ptr64 Void
   +0x438 EtwDataSource    : Ptr64 Void
   +0x440 PageDirectoryPte : Uint8B
   +0x448 ImageFilePointer : Ptr64 _FILE_OBJECT
   +0x450 ImageFileName    : [15] UChar // Process Name
   +0x45f PriorityClass    : UChar
   +0x460 SecurityPort     : Ptr64 Void
   +0x468 SeAuditProcessCreationInfo : _SE_AUDIT_PROCESS_CREATION_INFO
   +0x470 JobLinks         : _LIST_ENTRY
   +0x480 HighestUserAddress : Ptr64 Void
   +0x488 ThreadListHead   : _LIST_ENTRY
   +0x498 ActiveThreads    : Uint4B
   +0x49c ImagePathHash    : Uint4B
   +0x4a0 DefaultHardErrorProcessing : Uint4B
   +0x4a4 LastThreadExitStatus : Int4B
   +0x4a8 PrefetchTrace    : _EX_FAST_REF
   +0x4b0 LockedPagesList  : Ptr64 Void
   +0x4b8 ReadOperationCount : _LARGE_INTEGER
   +0x4c0 WriteOperationCount : _LARGE_INTEGER
   +0x4c8 OtherOperationCount : _LARGE_INTEGER
   +0x4d0 ReadTransferCount : _LARGE_INTEGER
   +0x4d8 WriteTransferCount : _LARGE_INTEGER
   +0x4e0 OtherTransferCount : _LARGE_INTEGER
   +0x4e8 CommitChargeLimit : Uint8B
   +0x4f0 CommitCharge     : Uint8B
   +0x4f8 CommitChargePeak : Uint8B
   +0x500 Vm               : _MMSUPPORT_FULL
   +0x640 MmProcessLinks   : _LIST_ENTRY
   +0x650 ModifiedPageCount : Uint4B
   +0x654 ExitStatus       : Int4B
   +0x658 VadRoot          : _RTL_AVL_TREE
   +0x660 VadHint          : Ptr64 Void
   +0x668 VadCount         : Uint8B
   +0x670 VadPhysicalPages : Uint8B
   +0x678 VadPhysicalPagesLimit : Uint8B
   +0x680 AlpcContext      : _ALPC_PROCESS_CONTEXT
   +0x6a0 TimerResolutionLink : _LIST_ENTRY
   +0x6b0 TimerResolutionStackRecord : Ptr64 _PO_DIAG_STACK_RECORD
   +0x6b8 RequestedTimerResolution : Uint4B
   +0x6bc SmallestTimerResolution : Uint4B
   +0x6c0 ExitTime         : _LARGE_INTEGER
   +0x6c8 InvertedFunctionTable : Ptr64 _INVERTED_FUNCTION_TABLE
   +0x6d0 InvertedFunctionTableLock : _EX_PUSH_LOCK
   +0x6d8 ActiveThreadsHighWatermark : Uint4B
   +0x6dc LargePrivateVadCount : Uint4B
   +0x6e0 ThreadListLock   : _EX_PUSH_LOCK
   +0x6e8 WnfContext       : Ptr64 Void
   +0x6f0 ServerSilo       : Ptr64 _EJOB
   +0x6f8 SignatureLevel   : UChar
   +0x6f9 SectionSignatureLevel : UChar
   +0x6fa Protection       : _PS_PROTECTION
   +0x6fb HangCount        : Pos 0, 3 Bits
   +0x6fb GhostCount       : Pos 3, 3 Bits
   +0x6fb PrefilterException : Pos 6, 1 Bit
   +0x6fc Flags3           : Uint4B
   +0x6fc Minimal          : Pos 0, 1 Bit
   +0x6fc ReplacingPageRoot : Pos 1, 1 Bit
   +0x6fc Crashed          : Pos 2, 1 Bit
   +0x6fc JobVadsAreTracked : Pos 3, 1 Bit
   +0x6fc VadTrackingDisabled : Pos 4, 1 Bit
   +0x6fc AuxiliaryProcess : Pos 5, 1 Bit
   +0x6fc SubsystemProcess : Pos 6, 1 Bit
   +0x6fc IndirectCpuSets  : Pos 7, 1 Bit
   +0x6fc RelinquishedCommit : Pos 8, 1 Bit
   +0x6fc HighGraphicsPriority : Pos 9, 1 Bit
   +0x6fc CommitFailLogged : Pos 10, 1 Bit
   +0x6fc ReserveFailLogged : Pos 11, 1 Bit
   +0x6fc SystemProcess    : Pos 12, 1 Bit
   +0x6fc HideImageBaseAddresses : Pos 13, 1 Bit
   +0x6fc AddressPolicyFrozen : Pos 14, 1 Bit
   +0x6fc ProcessFirstResume : Pos 15, 1 Bit
   +0x6fc ForegroundExternal : Pos 16, 1 Bit
   +0x6fc ForegroundSystem : Pos 17, 1 Bit
   +0x6fc HighMemoryPriority : Pos 18, 1 Bit
   +0x6fc EnableProcessSuspendResumeLogging : Pos 19, 1 Bit
   +0x6fc EnableThreadSuspendResumeLogging : Pos 20, 1 Bit
   +0x6fc SecurityDomainChanged : Pos 21, 1 Bit
   +0x6fc SecurityFreezeComplete : Pos 22, 1 Bit
   +0x6fc VmProcessorHost  : Pos 23, 1 Bit
   +0x700 DeviceAsid       : Int4B
   +0x708 SvmData          : Ptr64 Void
   +0x710 SvmProcessLock   : _EX_PUSH_LOCK
   +0x718 SvmLock          : Uint8B
   +0x720 SvmProcessDeviceListHead : _LIST_ENTRY
   +0x730 LastFreezeInterruptTime : Uint8B
   +0x738 DiskCounters     : Ptr64 _PROCESS_DISK_COUNTERS
   +0x740 PicoContext      : Ptr64 Void
   +0x748 EnclaveTable     : Ptr64 Void
   +0x750 EnclaveNumber    : Uint8B
   +0x758 EnclaveLock      : _EX_PUSH_LOCK
   +0x760 HighPriorityFaultsAllowed : Uint4B
   +0x768 EnergyContext    : Ptr64 _PO_PROCESS_ENERGY_CONTEXT
   +0x770 VmContext        : Ptr64 Void
   +0x778 SequenceNumber   : Uint8B
   +0x780 CreateInterruptTime : Uint8B
   +0x788 CreateUnbiasedInterruptTime : Uint8B
   +0x790 TotalUnbiasedFrozenTime : Uint8B
   +0x798 LastAppStateUpdateTime : Uint8B
   +0x7a0 LastAppStateUptime : Pos 0, 61 Bits
   +0x7a0 LastAppState     : Pos 61, 3 Bits
   +0x7a8 SharedCommitCharge : Uint8B
   +0x7b0 SharedCommitLock : _EX_PUSH_LOCK
   +0x7b8 SharedCommitLinks : _LIST_ENTRY
   +0x7c8 AllowedCpuSets   : Uint8B
   +0x7d0 DefaultCpuSets   : Uint8B
   +0x7c8 AllowedCpuSetsIndirect : Ptr64 Uint8B
   +0x7d0 DefaultCpuSetsIndirect : Ptr64 Uint8B
   +0x7d8 DiskIoAttribution : Ptr64 Void
   +0x7e0 DxgProcess       : Ptr64 Void
   +0x7e8 Win32KFilterSet  : Uint4B
   +0x7f0 ProcessTimerDelay : _PS_INTERLOCKED_TIMER_DELAY_VALUES
   +0x7f8 KTimerSets       : Uint4B
   +0x7fc KTimer2Sets      : Uint4B
   +0x800 ThreadTimerSets  : Uint4B
   +0x808 VirtualTimerListLock : Uint8B
   +0x810 VirtualTimerListHead : _LIST_ENTRY
   +0x820 WakeChannel      : _WNF_STATE_NAME
   +0x820 WakeInfo         : _PS_PROCESS_WAKE_INFORMATION
   +0x850 MitigationFlags  : Uint4B
   +0x850 MitigationFlagsValues : <anonymous-tag>
   +0x854 MitigationFlags2 : Uint4B
   +0x854 MitigationFlags2Values : <anonymous-tag>
   +0x858 PartitionObject  : Ptr64 Void
   +0x860 SecurityDomain   : Uint8B
   +0x868 ParentSecurityDomain : Uint8B
   +0x870 CoverageSamplerContext : Ptr64 Void
   +0x878 MmHotPatchContext : Ptr64 Void
```

### [#] Youtube
<iframe width="560" height="315" src="https://www.youtube.com/embed/Z8WT-VsV94M" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

# # Reference

1. MSDN
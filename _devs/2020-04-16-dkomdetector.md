---
layout: post
title: DKOM Detector Driver
tags:
- Windows
- Development
- Kernel
---
# # Introduce
기존에 포스팅 했던 ["Process Hide Driver"](https://holi4m.github.io/windows/2019/07/11/ProcessHideDriver/) 을 탐지하는 드라이버를 개발하였습니다.
ProcessListEntry와 ActiveProcessLinks를 이용하였습니다.

# # Testbed
> Windows Driver Kit 10.0.18362.1

### [#] Driver.c
드라이버 소스코드입니다.  
포스팅 작성 날짜 기준으로 Windows 10 모든 빌드를 지원합니다.  
Windows 10 빌드를 먼저 확인하고 오프셋을 전달하여  
Windows 10 18362 빌드까지는 정상적으로 구동합니다.  
향후 좀 더 유니버셜한 드라이버로 개발하도록 하겠습니다.  

```cpp
/*
[#]Holiam DKOM Detector
Using ActiveProcessLinks, ProcessListEntry
Windows 10 Universal (20-04-16)
BLOG = Holi4m.github.io
FeedBack = h01i4m
*/

#include <ntifs.h>

OSVERSIONINFOW osviThis;
BOOLEAN bPLEFlag = FALSE, bAPLFlag = FALSE;
int iPLECount = 0, iAPLCount = 0;
typedef unsigned long DWORD; // define DWORD

DWORD GetAPLOffset() // Get ActiveProcessLinks Offset Inside _EPROCESS Structure Func
{
	RtlGetVersion(&osviThis);
	if (osviThis.dwBuildNumber >= 18362)
	{
		return 0x2F0;
	}
	else if (osviThis.dwBuildNumber >= 10586)
	{
		return 0x2E8;
	}
	else
	{
		return 0;
	}

}

DWORD GetPLEOffset()  // Get ProcessListEntry Offset inside _KPROCESS Structure Func
{
	RtlGetVersion(&osviThis);
	if (osviThis.dwBuildNumber >= 18362)
	{
		return 0x248;
	}
	else if (osviThis.dwBuildNumber >= 10586)
	{
		return 0x240;
	}
	else
	{
		return 0;
	}

}

DWORD GetIFNOffset()  Get ImageFileName Offset Inside _EPROCESS Structure Func
{
	RtlGetVersion(&osviThis);
	if (osviThis.dwBuildNumber == 10240)
	{
		return 0x448;
	}
	else if (osviThis.dwBuildNumber > 10240)
	{
		return 0x450;
	}
	else
	{
		return 0;
	}
}


VOID Unload(_In_ PDRIVER_OBJECT pDriverObject)
{
	UNREFERENCED_PARAMETER(pDriverObject);
	DbgPrint("Driver Unload\n");
}

NTSTATUS DriverEntry(
	_In_ PDRIVER_OBJECT pDriverObject,
	_In_ PUNICODE_STRING pRegistryPath
)
{
	UNREFERENCED_PARAMETER(pDriverObject);
	UNREFERENCED_PARAMETER(pRegistryPath);
	pDriverObject->DriverUnload = Unload;

	DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[#] Holiam Detector Driver\n");
	
	PEPROCESS kpProcess = NULL;
	PLIST_ENTRY plPLE = NULL, plAPL = NULL, plPLEHead = NULL, plAPLHead = NULL;
	DWORD pPLE = 0, pAPL = 0, pIFN = 0;
	kpProcess = PsGetCurrentProcess();
	pPLE = GetPLEOffset();
	pAPL = GetAPLOffset();
	pIFN = GetIFNOffset();
	plPLE = (PLIST_ENTRY)((PCHAR)kpProcess + pPLE);
	plAPL = (PLIST_ENTRY)((PCHAR)kpProcess + pAPL);
	plAPLHead = plAPL->Blink;
	plPLEHead = plPLE->Blink;


	while (1)
	{
		if ((PCHAR)plAPL->Flink - (PCHAR)pAPL != (PCHAR)plPLE->Flink - (PCHAR)pPLE && (PCHAR)(plAPL->Flink) - (PCHAR)pAPL == (PCHAR)(plPLE->Flink->Flink) - (PCHAR)pPLE)
		{
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[APL Losted Detected!]\n");
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[PID is %x, PEPROCESS is %p", PsGetProcessId(((PEPROCESS)((PCHAR)plPLE->Flink - (PCHAR)pPLE))), ((PEPROCESS)((PCHAR)plPLE->Flink - (PCHAR)pPLE)));
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, ", ImageFIleName is %s]\n", ((PEPROCESS)((PCHAR)plPLE->Flink - (PCHAR)pPLE + (PCHAR)pIFN)));
			plPLE = plPLE->Flink;
			iPLECount++;
		}
		else if ((PCHAR)plAPL->Flink - (PCHAR)pAPL != (PCHAR)plPLE->Flink - (PCHAR)pPLE && (PCHAR)(plAPL->Flink->Flink) - (PCHAR)pAPL == (PCHAR)(plPLE->Flink) - (PCHAR)pPLE)
		{
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[PLE Losted Detected!]\n");
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[PID is %x, PEPROCESS is %p", PsGetProcessId(((PEPROCESS)((PCHAR)plAPL->Flink - (PCHAR)pAPL))), ((PEPROCESS)((PCHAR)plAPL->Flink - (PCHAR)pAPL)));
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, ", ImageFIleName is %s]\n", ((PEPROCESS)((PCHAR)plAPL->Flink - (PCHAR)pAPL + (PCHAR)pIFN)));
			plAPL = plAPL->Flink;
			iAPLCount++;
		}

		if(plPLE->Flink != plPLEHead)
		{
			iPLECount++;
		}
		else
		{
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[plPLEHead END, PLECount = %d]\n", iPLECount);
			bPLEFlag = TRUE;
		}

		if (plAPL->Flink != plAPLHead)
		{
			iAPLCount++;
		}
		else
		{
			DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "[plAPLHead END, APLCount = %d]\n", iAPLCount);
			bAPLFlag = TRUE;
		}

		if (bPLEFlag == FALSE && bAPLFlag == FALSE)
		{
			plAPL = plAPL->Flink;
			plPLE = plPLE->Flink;
		}
		else if (bPLEFlag == TRUE && bAPLFlag == FALSE)
		{
			plAPL = plAPL->Flink;
		}
		else if (bPLEFlag == FALSE && bAPLFlag == TRUE)
		{
			plPLE = plPLE->Flink;
		}
		else if (bPLEFlag == TRUE && bAPLFlag == TRUE)
		{
			break;
		}
	}	

	return STATUS_SUCCESS;
}
```

### [#] EPROCESS, KPROCESS Structure
드라이버 개발 시 사용한 구조체 오프셋입니다.  
Windows 10 1511 build부터 "_File_Object" 구조체 타입인 ImageFilePointer 가 추가되었습니다.  
저는 ImageFileName 오프셋만 사용하였습니다.

![1](http://holi4m.github.io/postimage/2020-04-17-dkomdetector/1.png)

## userinit.exe
userinit.exe는 Windows Native Application중 하나로써   
winlogon.exe가 실행 후 userinit.exe는 Explorer.exe를 실행하고 종료됩니다.  
Windows Native Application 이지만 종료되었는데 EPROCESS 구조체는 왜 남아있는지 모르겠습니다.  
아시는 분은 피드백 부탁드립니다.  

### [#] Youtube
<iframe width="560" height="315" src="https://www.youtube.com/embed/MpkZU9wqEcM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

# # Reference
1. [MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/)
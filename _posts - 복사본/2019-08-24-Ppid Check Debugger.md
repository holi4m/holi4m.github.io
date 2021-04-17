---
title: Debugger detector using PPID
date: 2019-08-24
categories:
- Windows
tags:
- Windows
- Dev
---

# # Introduce
안티 디버깅 기법 중 하나인 PPID를 이용하여 디버거를 탐지하는 기능을 구현 해 보았습니다.

# # PPID(Parent Process ID)
자신을 실행시킨 프로세스의 PID를 의미하며  
"_PROCESS_BASIC_INFORMATION" 구조체는 자신의 PID와 부모의 PID를 포함하고 있습니다.  

```cpp
typedef struct _PROCESS_BASIC_INFORMATION {
	NTSTATUS ExitStatus;
	PVOID PebBaseAddress;
	PVOID AffinityMask;
	PVOID BasePriority;
	ULONG_PTR UniqueProcessId;
	ULONG_PTR ParentProcessId;
} PROCESS_BASIC_INFORMATION, *PPROCESS_BASIC_INFORMATION;
```

# # How to
공격자가 Attach가 아닌 Debugger로 특정 프로그램을 실행 시키면 해당 프로그램은 PPID를 Debugger 의 PID를 가지게 됩니다.  

일반적으로 유저가 실행하는 프로그램은 대부분의 경우 Explorer가 부모 프로세스이나  
Launcher나 Anti-Cheat Solution을 사용하는 경우 별도의 프로세스가 먼저 실행됩니다.  

>Explorer - Steam - Game  

이럴 경우 실행 된 Game의 경우 PPID는 Steam의 PID가 되며, Game의 경우 자기 자신의 PPID를 검증하여 정상 실행 되었는지를 판단합니다.    


### [#] CheckPPID.cpp
부모 프로세스가 Explorer.exe가 아닐 경우 탐지하는 소스 코드입니다.

```cpp
#include <windows.h>
#include <stdio.h>
#include <psapi.h>

typedef struct _PROCESS_BASIC_INFORMATION {
    NTSTATUS ExitStatus;
    PVOID PebBaseAddress;
    PVOID AffinityMask;
    PVOID BasePriority;
    ULONG_PTR UniqueProcessId;
    ULONG_PTR ParentProcessId;
} PROCESS_BASIC_INFORMATION, * PPROCESS_BASIC_INFORMATION; //Define _PROCESS_BASIC_INFORMATION

typedef NTSTATUS(WINAPI* qip)(HANDLE, UINT, PVOID, ULONG, PULONG); //Define NtQueryInformationProcess

DWORD ParentProcessPID() {
    PROCESS_BASIC_INFORMATION pbi;
    ZeroMemory(&pbi, 48);

    qip proc = (qip)GetProcAddress(GetModuleHandle("NTDLL.DLL"), "NtQueryInformationProcess");

    NTSTATUS stat = proc(GetCurrentProcess(), 0, &pbi, 48, 0);
    if (stat == 0) {
        return pbi.ParentProcessId;
    }
    else {
        printf("NtQueryInformationProcess ErrorCode = %d\n", GetLastError());
    }
    return 0;
};

BOOL CkPpid(ULONG_PTR ULPpid) {
    char PpidName[100] = { NULL };
    ULONG_PTR Ppid = NULL;
    HANDLE hPpid = NULL;
    const char Ppname[13] = "Explorer.EXE";
    int Retlen = NULL;

    Ppid = ULPpid;
    hPpid = OpenProcess(0x410, FALSE, Ppid); // 0x410 = PROCESS_QUERY_INFORMATION (0x0400) & PROCESS_VM_READ (0x0010) or PROCESS_ALL_ACCESS

    Retlen = GetModuleBaseName(hPpid, NULL, PpidName, sizeof(PpidName));
    if (Retlen != 0) {
        printf("PpName = %s\n", Ppname);
        printf("PpidName = %s\n", PpidName);
        if (*Ppname == *PpidName) {
            return TRUE;
        }
    }
    else {
        printf("GetModuleBaseName ErrorCode = %d\n", GetLastError());
        return FALSE;
    }


    return FALSE;
}

int main() {

    BOOL CkDbg;
    CkDbg = CkPpid(ParentProcessPID());
    if (CkDbg == FALSE) 
    {
        printf("Ppid is Not Explorer. \n");
    }
    else 
    {
        printf("Ppid is Explorer. \n");
    }
    return 0;
	Sleep(10000);
}

```

### [#] 실행 결과

```
Check Parent Process = Explorer.EXE
Running Parent Process = VsDebugConsole.exe
Ppid is Not Explorer.

Check Parent Process = Explorer.EXE
Running Parent Process = Explorer.EXE
Ppid is Explorer.
```

## ## How to ScyllaHide is Bypass PPID?
해당 코드를 짜면서 더 찾아 봤던 것은 x64dbg Plugin인 ScyllaHide는 어떻게 PPID를 우회하는가 였는데  
PPID Bypass 부분 소스코드를 확인하면 

```
else if (ProcessInformationClass == ProcessBasicInformation)
{
	BACKUP_RETURNLENGTH()
	
	((PPROCESS_BASIC_INFORMATION)ProcessInformation)->InheritedFromUniqueProcessId = ULongToHandle(GetExplorerProcessId()));
	
	RESTORE_RETURNLENGTH()
}
```

NtQueryInformationProcess 의 2번째 Argument인 ProcessInformationClass가  
ProcessBasicInformation(0)일 경우 Explorer의 PID로 변조하는것을 확인하였습니다.

ScyllaHide는 이렇게 Bypass한다. 정도로 알고 있으면 될 듯 합니다.

# # Reference

1. [[MSDN]](https://docs.microsoft.com/en-us/windows/win32/api/winternl/nf-winternl-ntqueryinformationprocess)
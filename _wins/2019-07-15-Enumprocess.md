---
layout: post
title: "EnumProcesses & GetNativeSystemInfo"
tags:
- Windows
- Kernel
---
# # Introduce
Process Hide를 코딩하고 Process Hide를 탐지할 방법을 찾아보다 알게되어 테스트 해봤습니다.  

# # EnumProcesses API
EnumProcesses API는 현재 실행중인 프로세스 리스트를 배열에 담아주는 API입니다.

### [#] EnumProcess Func
```cpp
BOOL WINAPI EnumProcesses(
    _Out_writes_bytes_(cb) DWORD* lpidProcess, //pid 배열 포인터
    _In_ DWORD cb, // 배열 사이즈 
    _Out_ LPDWORD lpcbNeeded // 리턴된 pid 배열의 바이트 수
    );
```

### [#] EnumProcesses.cpp
실행중인 프로세스의 PID를 배열에 입력받아 Process Name, PID를 출력해주는 소스입니다.  
PID로 Process Name을 못찾을 때는 <unknown>으로 출력합니다.  

```cpp
#include <windows.h>
#include <stdio.h>
#include <tchar.h>
#include <psapi.h>
// To ensure correct resolution of symbols, add Psapi.lib to TARGETLIBS
// and compile with -DPSAPI_VERSION=1
void PrintProcessNameAndID(DWORD processID){
	TCHAR szProcessName[MAX_PATH] = TEXT("<unknown>");
	// Get a handle to the process.
	HANDLE hProcess = OpenProcess(PROCESS_QUERY_INFORMATION |
	PROCESS_VM_READ,
	FALSE, processID);
	// Get the process name.
	if (NULL != hProcess){
		HMODULE hMod;
		DWORD cbNeeded;
		if (EnumProcessModules(hProcess, &hMod, sizeof(hMod),&cbNeeded)){
			GetModuleBaseName(hProcess, hMod, szProcessName,
			sizeof(szProcessName) / sizeof(TCHAR));}
		}
	// Print the process name and identifier.
	_tprintf(TEXT("%s  (PID: %u)\n"), szProcessName, processID);
	// Release the handle to the process.
	CloseHandle(hProcess);
}

int main(void)
{
	// Get the list of process identifiers.
	DWORD aProcesses[1024], cbNeeded, cProcesses;
	unsigned int i;
		if (!EnumProcesses(aProcesses, sizeof(aProcesses), &cbNeeded)){
			return 1;
		}
	// Calculate how many process identifiers were returned.
	cProcesses = cbNeeded / sizeof(DWORD);
	printf("sizeof(aProcesses) = %zd\ncbNeeded = %ld\ncProcesses=%ld", sizeof(aProcesses), cbNeeded,cProcesses);
	// Print the name and process identifier for each process.
		for (i = 0; i < cProcesses; i++)	{
			if (aProcesses[i] != 0){
				PrintProcessNameAndID(aProcesses[i]);
			}
		}
	return 0;
}
```
해당 소스를 컴파일해서 x64dbg로 확인해보면 EnumProcesses > Kernel32.K32EnumProcesses > Kernelbase.K32EnumProcesses순으로 호출하며  
최종적으로 시스템 정보를 RtlGetNativeSystemInformation함수를 이용해 받아옵니다.
```
mov rax,rsp
mov qword ptr ds:[rax+8],rbx
mov qword ptr ds:[rax+10],rsi
mov qword ptr ds:[rax+18],rdi
push r12
push r14
push r15
sub rsp,30
mov r15,r8
mov r14d,edx
mov r12,rcx
and dword ptr ds:[rax+20],0
mov edi,10000
mov rax,qword ptr gs:[60]
mov rsi,qword ptr ds:[rax+30]
mov qword ptr ss:[rsp+28],rsi
mov r8d,edi
xor edx,edx
mov rcx,rsi
call qword ptr ds:[<&RtlAllocateHeap>]
nop dword ptr ds:[rax+rax],eax
mov rbx,rax
mov qword ptr ss:[rsp+20],rax
test rax,rax
je kernelbase.7FFF582AD9F2
lea r9,qword ptr ss:[rsp+68]
mov r8d,edi
mov rdx,rax
mov ecx,5
call qword ptr ds:[<&RtlGetNativeSystemInformation>]
nop dword ptr ds:[rax+rax],eax
mov edi,eax
cmp eax,C0000004
jne kernelbase.7FFF5826E6B1
mov r8,rbx
xor edx,edx
mov rcx,rsi
call qword ptr ds:[<&RtlFreeHeap>]
nop dword ptr ds:[rax+rax],eax
mov edi,dword ptr ss:[rsp+68]
jmp kernelbase.7FFF5826E64D
test edi,edi
js kernelbase.7FFF582AD9CC
xor r9d,r9d
shr r14d,2
xor edx,edx
mov r8d,r9d
cmp edx,r14d
jae kernelbase.7FFF5826E711
mov ecx,dword ptr ds:[r9+rbx+50]
mov dword ptr ds:[r12+rdx*4],ecx
jmp kernelbase.7FFF5826E70F
mov ebx,eax
mov r8,qword ptr ss:[rsp+20]
xor edx,edx
mov rcx,qword ptr ss:[rsp+28]
call qword ptr ds:[<&RtlFreeHeap>]
nop dword ptr ds:[rax+rax],eax
mov ecx,ebx
call qword ptr ds:[<&RtlNtStatusToDosError>]
nop dword ptr ds:[rax+rax],eax
mov ecx,eax
call qword ptr ds:[<&RtlRestoreLastWin32Error>]
nop dword ptr ds:[rax+rax],eax
xor eax,eax
jmp kernelbase.7FFF5826E779
inc edx
add r9d,dword ptr ds:[r9+rbx]
cmp dword ptr ds:[r8+rbx],0
jne kernelbase.7FFF5826E6C2
mov eax,edx
shl eax,2
mov dword ptr ds:[r15],eax
jmp kernelbase.7FFF5826E760
mov ebx,eax
mov r8,qword ptr ss:[rsp+20]
xor edx,edx
mov rcx,qword ptr ss:[rsp+28]
call qword ptr ds:[<&RtlFreeHeap>]
nop dword ptr ds:[rax+rax],eax
mov ecx,ebx
call qword ptr ds:[<&RtlNtStatusToDosError>]
nop dword ptr ds:[rax+rax],eax
mov ecx,eax
call qword ptr ds:[<&RtlRestoreLastWin32Error>]
nop dword ptr ds:[rax+rax],eax
xor eax,eax
jmp kernelbase.7FFF5826E779
mov r8,rbx
xor edx,edx
mov rcx,rsi
call qword ptr ds:[<&RtlFreeHeap>]
nop dword ptr ds:[rax+rax],eax
mov eax,1
mov rbx,qword ptr ss:[rsp+50]
mov rsi,qword ptr ss:[rsp+58]
mov rdi,qword ptr ss:[rsp+60]
add rsp,30
pop r15
pop r14
pop r12
ret 
```
RtlGetNativeSystemInformation은 Syscall 인자(RAX)를 36을 전달합니다.
```
mov r10,rcx
mov eax,36
test byte ptr ds:[7FFE0308],1
jne ntdll.7FFF5A81C785
syscall 
ret 
int 2E
ret 
```
Syscall 인자 36의 경우 Windows Syscall Table을 확인해보면 Windows 10 모든 빌드에서  
NtQuerySystemInformation을 호출하며  
Syscall에 사용되는 EnumProcesses의 Argument는 아래와 같습니다.  
```
1: rcx 0000000000000005 // SystemProcessInformation
2: rdx 00000250888525F0 
3: r8 0000000000010000 
4: r9 000000C035DCE3C8 
5: [rsp+20] 00000250888525F0 
```

rcx에 System_Information_Class 값이 0x05(SystemProcessInformation)이 전달되어  
시스템에 존재하는 프로세스의 정보를 받아 오게 됩니다.

# # GetNativeSystemInfo API
GetNativeSystemInfo API는 현재 시스템의 정보를 가지고 오는 API입니다.  
x64에서는 GetNativeSysteminfo이며, x32에서는 GetSystemInfo입니다.  
EnumProcess와 동일하게 RtlGetNativeSystemInformation를 이용할 것 같아 분석하였습니다.

### [#] GetNativeSystemInfo
```cpp
WINBASEAPI VOID WINAPI GetNativeSystemInfo(
    _Out_ LPSYSTEM_INFO lpSystemInfo
    );
```

### [#] GetNativeSystemInfo.cpp
함수 호출만 확인하기 위해 간단하게 구현하였습니다.  
```cpp
#include <Windows.h>
int main()
{
	SYSTEM_INFO sInfo;
	GetNativeSystemInfo(&sInfo);
	return 0;
}
```

GetNativeSystemInfo에서 시스템 정보를 받아오는 System_Info 구조체입니다.  

```
typedef struct _SYSTEM_INFO {
  union {
    DWORD dwOemId;
	
    struct {
      WORD wProcessorArchitecture;
      WORD wReserved;
    } DUMMYSTRUCTNAME;
  } DUMMYUNIONNAME;
  DWORD     dwPageSize;
  LPVOID    lpMinimumApplicationAddress;
  LPVOID    lpMaximumApplicationAddress;
  DWORD_PTR dwActiveProcessorMask;
  DWORD     dwNumberOfProcessors;
  DWORD     dwProcessorType;
  DWORD     dwAllocationGranularity;
  WORD      wProcessorLevel;
  WORD      wProcessorRevision;
} SYSTEM_INFO, *LPSYSTEM_INFO;
```

해당 소스를 컴파일해 x64dbg로 확인하면 해당 함수는 직접적으로 RtlGetNativeSystemInformation를 호출하며
EnumProcesses와 동일하게 Syscall NtQuerySystemInformation을 호출합니다.
```
push rbx
sub rsp,80
mov rax,qword ptr ds:[<__security_cookie>]
xor rax,rsp
mov qword ptr ss:[rsp+70],rax
xor r9d,r9d
lea rdx,qword ptr ss:[rsp+30]
mov rbx,rcx
xor ecx,ecx
lea r8d,qword ptr ds:[r9+40]
call qword ptr ds:[<&ZwQuerySystemInformation>]
nop dword ptr ds:[rax+rax],eax
test eax,eax
js kernelbase.7FFF5823A42B
xor r9d,r9d
lea rdx,qword ptr ss:[rsp+20]
lea r8d,qword ptr ds:[r9+C]
lea ecx,qword ptr ds:[r9+1]
call qword ptr ds:[<&ZwQuerySystemInformation>]
nop dword ptr ds:[rax+rax],eax
test eax,eax
js kernelbase.7FFF5823A42B
mov r8,rbx
lea rdx,qword ptr ss:[rsp+20]
lea rcx,qword ptr ss:[rsp+30]
call <kernelbase.GetSystemInfoInternal>
mov rcx,qword ptr ss:[rsp+70]
xor rcx,rsp
call <kernelbase.__security_check_cookie>
add rsp,80
pop rbx
ret 
```
각각 호출하는 ZwQuerySystemInformation의 Argument는 아래와 같습니다.  

1번째 ZwQuerySystemInformation Argument
```
1: rcx 0000000000000000 // SystemBasicInformation
2: rdx 000000801E0FF7B0 
3: r8 0000000000000040 
4: r9 0000000000000000 
5: [rsp+20] 0030003400300030 
```

2번째 ZwQuerySystemInformation Argument
```
1: rcx 0000000000000001 // SystemProcessorInformation
2: rdx 000000801E0FF7A0 
3: r8 000000000000000C 
4: r9 0000000000000000 
5: [rsp+20] 0030003400300030 
```

2번의 ZwQuerySystemInformation을 시스템과 CPU의 정보를 확인하고
Kernalbase.GetSystemInfoInternal 함수 내부에서 Memset 이후 NtQueryInformationProcess를 호출합니다.

NtSystemInformationProcess Argument
```
1: rcx FFFFFFFFFFFFFFFF 
2: rdx 0000000000000025 
3: r8 000000801E0FF730 
4: r9 0000000000000040 
5: [rsp+20] 0000000000000000 
```
NtQueryInformationProcess 호출 후 sInfo에 시스템 정보가 담겨 있습니다.

# # Conclusion
ActiveProcessLinks를 끊은 Process는 EnumProcesses로 탐지 되지 않습니다.  
EnumProcesses가 "_Eprocess"의 ProcessActiveLinks를 이용하여 프로세스를 찾기 때문입니다.  
차후 왜 EnumProcess를 이용하여 탐지가 되지 않는지 NtQuerySystemInformation을 분석 해보도록 하겠습니다.  

# # Reference

1. [Windows SysCall Table (X64)](https://j00ru.vexillium.org/syscalls/nt/64/)
1. [System_Information_Class Table](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/api/ex/sysinfo/class.htm?tx=65,67&ts=0,268)
---
layout: post
title: SetPrivilege Function & Access Token Privilege
tags:
- Windows
- Dev
---

# # Introduce
SetPrivilege 함수는 주로 Injector에서 많이 사용되고 Debugger와 Cheat Engine등의 Debugging을 수행하는 프로그램에 쓰일 것 같은 함수입니다.  
해당 프로세스의 사용자 엑세스 토큰의(Access Token) 특권을 이용하여 본인이 가지고 있지 않는 권한을 이용할 수 있습니다.  
또한 같은 사용자라도 해당 프로세스가 실행 된 권한에 따라 사용할 수 있는 특권에도 차이가 있습니다.  

# # Access Token
엑세스 토큰이란 Windows에서 로그인 할 때 생성되며 객체(Object)가 객체(Object)에 접근하기 위해 사용되는 접근 권한에 대한 정보입니다.

# # SetPrivilege
특권 권한 설정을 할 수 있는 함수로써 리버싱 핵심원리, MSDN에서 해당 소스를 제공하고 있습니다.

### [#] SetPrivilege.cpp 
SetPrivilege와 연관되어 있는 함수, 구조체입니다.

+ SetPrivilege Function  

```cpp
#include <windows.h>
#include <stdio.h>
#pragma comment(lib, "cmcfg32.lib")

BOOL SetPrivilege(
    HANDLE hToken,          // access token handle
    LPCTSTR lpszPrivilege,  // name of privilege to enable/disable
    BOOL bEnablePrivilege   // to enable or disable privilege
    ) 
{
    TOKEN_PRIVILEGES tp;
    LUID luid;

    if ( !LookupPrivilegeValue( 
            NULL,            // lookup privilege on local system
            lpszPrivilege,   // privilege to lookup 
            &luid ) )        // receives LUID of privilege
    {
        printf("LookupPrivilegeValue error: %u\n", GetLastError() ); 
        return FALSE; 
    }

    tp.PrivilegeCount = 1;
    tp.Privileges[0].Luid = luid;
    if (bEnablePrivilege)
        tp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;
    else
        tp.Privileges[0].Attributes = 0;

    // Enable the privilege or disable all privileges.

    if ( !AdjustTokenPrivileges(
           hToken, 
           FALSE, 
           &tp, 
           sizeof(TOKEN_PRIVILEGES), 
           (PTOKEN_PRIVILEGES) NULL, 
           (PDWORD) NULL) )
    { 
          printf("AdjustTokenPrivileges error: %u\n", GetLastError() ); 
          return FALSE; 
    } 

    if (GetLastError() == ERROR_NOT_ALL_ASSIGNED)

    {
          printf("The token does not have the specified privilege. \n");
          return FALSE;
    } 

    return TRUE;
}
```
- Token_Privileges Structure  

```cpp
typedef struct _TOKEN_PRIVILEGES 
{
	DWORD               PrivilegeCount; // 배열 갯수
	LUID_AND_ATTRIBUTES Privileges[ANYSIZE_ARRAY]; // luid,attribute 구조체
} TOKEN_PRIVILEGES, *PTOKEN_PRIVILEGES;

//LUID_AND_ATTRIBUTES 구조체 구조

typedef struct _LUID_AND_ATTRIBUTES 
{ 
	LUID  Luid; // LUID
	DWORD Attributes; // 특권 상태
} LUID_AND_ATTRIBUTES, *PLUID_AND_ATTRIBUTES;
```

- AdjustTokenPrivileges Function

```cpp
BOOL AdjustTokenPrivileges
(
  HANDLE            TokenHandle,
  BOOL              DisableAllPrivileges,
  PTOKEN_PRIVILEGES NewState,
  DWORD             BufferLength,
  PTOKEN_PRIVILEGES PreviousState,
  PDWORD            ReturnLength
);
```

- Privilege Lockup Table

```cpp
#define SE_CREATE_TOKEN_NAME              TEXT("SeCreateTokenPrivilege")
#define SE_ASSIGNPRIMARYTOKEN_NAME        TEXT("SeAssignPrimaryTokenPrivilege")
#define SE_LOCK_MEMORY_NAME               TEXT("SeLockMemoryPrivilege")
#define SE_INCREASE_QUOTA_NAME            TEXT("SeIncreaseQuotaPrivilege")
#define SE_UNSOLICITED_INPUT_NAME         TEXT("SeUnsolicitedInputPrivilege")
#define SE_MACHINE_ACCOUNT_NAME           TEXT("SeMachineAccountPrivilege")
#define SE_TCB_NAME                       TEXT("SeTcbPrivilege")
#define SE_SECURITY_NAME                  TEXT("SeSecurityPrivilege")
#define SE_TAKE_OWNERSHIP_NAME            TEXT("SeTakeOwnershipPrivilege")
#define SE_LOAD_DRIVER_NAME               TEXT("SeLoadDriverPrivilege")
#define SE_SYSTEM_PROFILE_NAME            TEXT("SeSystemProfilePrivilege")
#define SE_SYSTEMTIME_NAME                TEXT("SeSystemtimePrivilege")
#define SE_PROF_SINGLE_PROCESS_NAME       TEXT("SeProfileSingleProcessPrivilege")
#define SE_INC_BASE_PRIORITY_NAME         TEXT("SeIncreaseBasePriorityPrivilege")
#define SE_CREATE_PAGEFILE_NAME           TEXT("SeCreatePagefilePrivilege")
#define SE_CREATE_PERMANENT_NAME          TEXT("SeCreatePermanentPrivilege")
#define SE_BACKUP_NAME                    TEXT("SeBackupPrivilege")
#define SE_RESTORE_NAME                   TEXT("SeRestorePrivilege")
#define SE_SHUTDOWN_NAME                  TEXT("SeShutdownPrivilege")
#define SE_DEBUG_NAME                     TEXT("SeDebugPrivilege")
#define SE_AUDIT_NAME                     TEXT("SeAuditPrivilege")
#define SE_SYSTEM_ENVIRONMENT_NAME        TEXT("SeSystemEnvironmentPrivilege")
#define SE_CHANGE_NOTIFY_NAME             TEXT("SeChangeNotifyPrivilege")
#define SE_REMOTE_SHUTDOWN_NAME           TEXT("SeRemoteShutdownPrivilege")
#define SE_UNDOCK_NAME                    TEXT("SeUndockPrivilege")
#define SE_SYNC_AGENT_NAME                TEXT("SeSyncAgentPrivilege")
#define SE_ENABLE_DELEGATION_NAME         TEXT("SeEnableDelegationPrivilege")
#define SE_MANAGE_VOLUME_NAME             TEXT("SeManageVolumePrivilege")
#define SE_IMPERSONATE_NAME               TEXT("SeImpersonatePrivilege")
#define SE_CREATE_GLOBAL_NAME             TEXT("SeCreateGlobalPrivilege")
#define SE_TRUSTED_CREDMAN_ACCESS_NAME    TEXT("SeTrustedCredManAccessPrivilege")
#define SE_RELABEL_NAME                   TEXT("SeRelabelPrivilege")
#define SE_INC_WORKING_SET_NAME           TEXT("SeIncreaseWorkingSetPrivilege")
#define SE_TIME_ZONE_NAME                 TEXT("SeTimeZonePrivilege")
#define SE_CREATE_SYMBOLIC_LINK_NAME      TEXT("SeCreateSymbolicLinkPrivilege")
```

### [#] SetPrivilege 함수 분석

LookupPrivilegeValue 함수를 이용해서 특권이 적용된 LUID를 받아옵니다.
  
```cpp
#include <windows.h>
#include <stdio.h>
#pragma comment(lib, "cmcfg32.lib")

BOOL SetPrivilege(
    HANDLE hToken,          // access token handle
    LPCTSTR lpszPrivilege,  // name of privilege to enable/disable
    BOOL bEnablePrivilege   // to enable or disable privilege
    ) 
{
    TOKEN_PRIVILEGES tp;
    LUID luid;

    if ( !LookupPrivilegeValue( 
            NULL,            // lookup privilege on local system
            lpszPrivilege,   // privilege to lookup 
            &luid ) )        // receives LUID of privilege
    {
        printf("LookupPrivilegeValue error: %u\n", GetLastError() ); 
        return FALSE; 
    }

```
LookupPrivilegeValue함수를 통해 가져온 LUID를 tp.Luid 에 적용하고 특권을 Enabled 시킵니다.  

```cpp
    tp.PrivilegeCount = 1;
    tp.Privileges[0].Luid = luid;
    if (bEnablePrivilege)
        tp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;
    else
        tp.Privileges[0].Attributes = 0;
		
```

AdjustTokenPrivileges함수를 통해 엑세스 토큰에 특권을 적용시킵니다.  

```cpp
    if ( !AdjustTokenPrivileges(
           hToken, 
           FALSE, 
           &tp, 
           sizeof(TOKEN_PRIVILEGES), 
           (PTOKEN_PRIVILEGES) NULL, 
           (PDWORD) NULL) )
    { 
          printf("AdjustTokenPrivileges error: %u\n", GetLastError() ); 
          return FALSE; 
    } 

    if (GetLastError() == ERROR_NOT_ALL_ASSIGNED)

    {
          printf("The token does not have the specified privilege. \n");
          return FALSE;
    } 

    return TRUE;
}
```

## ## 프로세스 실행 권한에 따른 LUID 차이
ProcessHacker로 LUID를 확인하면 관리자 실행 명령 프롬프트(좌) 와 기본 유저 명령 프롬프트(우)의 특권이 다른 것을 확인 할 수 있습니다.

![1](http://holi4m.github.io/postimage/2019-06-06-Privilege/1.png)

## ## SetPrivilege 함수 테스트
테스트로는 앞서 포스팅했던 Code Injection 포스팅의 소스코드를 약간 수정하여 사용하였습니다. 

## ## User to Admin Injection
![2](http://holi4m.github.io/postimage/2019-06-06-Privilege/2.png)

유저 권한으로 Code Injection을 시도하면 토큰에 특권이 존재하지 않아 SetPrivilege 함수 에러가 발생하며,  
SetPrivilege 함수를 이용하지 않을 경우에는 권한이 없기 때문에 OpenProcess Err_code 5 (Access Denied) 에러가 발생 합니다.

## ## Admin to Admin Injection
![3](http://holi4m.github.io/postimage/2019-06-06-Privilege/3.png)

![4](http://holi4m.github.io/postimage/2019-06-06-Privilege/4.png)
관리자 권한의 경우 SetPrivilege 유무에 상관 없이 Injection이 성공하는 것을 확인하였습니다.  

## ## Admin to System Injection
![5](http://holi4m.github.io/postimage/2019-06-06-Privilege/5.png)

System 권한을 가진 svchost.exe를 대상으로 인젝션을 시도하였습니다..  

SetPrivilege를 이용 할 경우 정상적으로 프로세스의 핸들을 가져오며, 쓰레드까지 생성하나 메시지박스가 뜨지 않습니다.  

해당 부분에 대한 자세한 내용은 차후에 공부하여 더 알아보도록 하겠습니다.  

![6](http://holi4m.github.io/postimage/2019-06-06-Privilege/6.png)

SetPrivilege를 이용하지 않았을 경우 권한이 없어 프로세스 핸들을 가져오지 못합니다.

# # 마무리
Access Token의 개념을 이해하지 못했을 때 가장 이해가 안갔던 부분이 왜 Injector들은 SetPrivilege를 쓸까? 책에서 보고 직접 해보니 이해가 갔습니다.. 이해가 안가는 경우 꼭 해보길 바랍니다.  

Access Token에 대하여 더 이해하고 싶다면 윈도우즈 이터널의 보안 챕터 (5판기준 6장 / 7판기준 7장)를 읽어보는 것을 추천 드립니다.
# # Relation
1. [[Code Injection]]

# # Reference
1. [리버싱 핵심원리]]
2. [윈도우즈 이터널 ]
3. [[MSDN]](https://docs.microsoft.com/en-us/windows/win32/secauthz/access-tokens)
4. [[Blog]](https://tribal1012.tistory.com/215)
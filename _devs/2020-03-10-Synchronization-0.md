---
layout: post
title: Windows Synchronization_0x00
tags:
- Windows
---
# # Introduce
프로세스, DLL간의 통신 구간을 알아보다 Mutex, 임계 영역(Critical Section) 등을 보게 되어 해당 부분에 대해 재대로 공부하고 정리하게 되었습니다.

# # Synchronization
운영체제에서 특정 메모리에 프로세스, 스레드가 동시 접근을 할 때 메모리의 원자성이 깨지지 않도록 임계 영역에서 작업을 해주는 것을 말하며  

각각의 스레드 또는 프로세스가 임계 영역에서 작업을 끝내기 전까지 다른 스레드나 프로세스의 임계 영역 접근을 제어하는 것을 의미합니다.  

화장실을 예시로 많이 들며, 화장실 칸을 A가 사용 할 경우 B가 화장실에 들어가지 못하는 것을 생각하면 됩니다.  

## ## User Level Synchronization techniques
윈도우에서 제공하는 메모리 동기화 기법은 User Level과 Kernel Level로 나뉘어 지고, 각각 장단점이 존재합니다.  

유저 레벨에서 이루어지는 동기화는 커널 레벨까지 접근하지 않아 속도가 빠르다는 장점이 있습니다.  
그러나 커널 오브젝트를 생성하지 않고 프로세스 내부에서 돌아가기 때문에 프로세스 간 공유가 불가능합니다.  

- User Level
	- Critical Section
	- InterLocked Func
	- Etc ...

해당 포스팅에서는 User Level 의 Critical Section, InterLocked Func을 구현하여 Windows Synchronization을 테스트 하였습니다.  

### [#] CriticalSection.cpp
CriticalSection 함수를 적용한 소스입니다.  
Mutex와 비슷하며 CriticalSection 내부의 코드는 단일 스레드만 접근 할 수 있습니다.  

```cpp
#include <stdio.h>
#include <process.h>
#include <Windows.h>

int count;
int iloopnum = 0;
CRITICAL_SECTION hCriticalSection;

DWORD WINAPI SectionChildThreadProcedure(LPVOID arglp)
{
	DWORD dThreadId = GetCurrentThreadId();

	printf("Thread Id = 0x%08x Before Critical Section Count = %d\n", dThreadId, count);

	EnterCriticalSection(&hCriticalSection); // Enter Critical Section

	for (int n = 0; n < (int)arglp; n++)
	{
		
		count++;
		
	}
	LeaveCriticalSection(&hCriticalSection); // Leave Critical Section

	printf("Thread Id = 0x%08x After Critical Section Count = %d\n", dThreadId, count);

	return 0;
}

DWORD WINAPI NoSectionChildThreadProcedure(LPVOID arglp)
{
	DWORD dThreadId = GetCurrentThreadId();

	printf("Thread Id = 0x%08x Before Nothing Count = %d\n", dThreadId, count);

	for (int n = 0; n < (int)arglp; n++)
	{
		count++;
	}
	printf("Thread Id = 0x%08x After Nothing Count = %d\n", dThreadId, count);

	return 0;
}

int CreateThreadFunc(int argv)
{
	HANDLE hChildThread[3];
	DWORD dUnusedThreadId;

	int check = argv;

	InitializeCriticalSection(&hCriticalSection); // initalized Critical Section

	printf("Input Loop Num : ");
	scanf_s("%d", &iloopnum);

	if (check == 1)
	{
		hChildThread[0] = (HANDLE)CreateThread(NULL, 0, NoSectionChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[1] = (HANDLE)CreateThread(NULL, 0, NoSectionChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[2] = (HANDLE)CreateThread(NULL, 0, NoSectionChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
	}
	else if (check == 2)
	{
		hChildThread[0] = (HANDLE)CreateThread(NULL, 0, SectionChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[1] = (HANDLE)CreateThread(NULL, 0, SectionChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[2] = (HANDLE)CreateThread(NULL, 0, SectionChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
	}
	else
	{
		printf("check = %d Is illegal Value\n",check);
		return 0;
	}
	
	if (!hChildThread[0] || !hChildThread[1] || !hChildThread[2])
	{
		printf("Error CreateChildThread Error");
		return NULL;
	}

	DWORD dwResult = WaitForMultipleObjects(3, hChildThread, TRUE, INFINITE);

	if (dwResult != WAIT_OBJECT_0)
	{
		printf("Error WaitForMultipleObject()");
		return 0;
	}

	CloseHandle(hChildThread[0]);
	CloseHandle(hChildThread[1]);
	CloseHandle(hChildThread[2]);

	DeleteCriticalSection(&hCriticalSection); 

	return 0;

}

int main(void)
{
	int i = 0;
	while (1)
	{
		printf("1. Nothing\n2. Set Mutex\nInput Number : ");
		scanf_s("%d", &i);
		if (i == 1 || i == 2)
		{
			CreateThreadFunc(i);
			return 0;
		}

		else
		{
			printf("Input legal Value\n");
		}
	}

}
```

### [#] 실행 결과
실행 결과는 아래와 같습니다.
반복문 코드 영역이 단일 스레드만 접근이 가능하여 10000000단위로 쓰레드가 종료됩니다. 

```
1. Nothing
2. Set Mutex
Input Number : 1
Input Loop Num : 10000000
Thread Id = 0x000015e0 Before Nothing Count = 0
Thread Id = 0x00003410 Before Nothing Count = 0
Thread Id = 0x00003c4c Before Nothing Count = 0
Thread Id = 0x000015e0 After Nothing Count = 7315211
Thread Id = 0x00003410 After Nothing Count = 10523194
Thread Id = 0x00003c4c After Nothing Count = 10942567

1. Nothing
2. Set Mutex
Input Number : 2
Input Loop Num : 10000000
Thread Id = 0x00003518 Before Critical Section Count = 0
Thread Id = 0x00002f54 Before Critical Section Count = 0
Thread Id = 0x00003a2c Before Critical Section Count = 301601 // Thread 생성 시 이미 임계 영역에서 count++이 진행중입니다.
Thread Id = 0x00003518 After Critical Section Count = 10000000
Thread Id = 0x00002f54 After Critical Section Count = 20000000
Thread Id = 0x00003a2c After Critical Section Count = 30000000
```

### [#] InterLocked.cpp
InterLocked 계열 함수를 적용한 소스입니다.
Critical Section보다 더 단순하게 적용이 가능하며, 함수를 통해 임계 영역에서 변수 값을 증가시킵니다.

```cpp
#include <stdio.h>
#include <process.h>
#include <Windows.h>

int iloopnum;
LONG volatile count; // Define Interlocked Argument

DWORD WINAPI InterLockedChildThreadProcedure(LPVOID arglp)
{
	DWORD dThreadId = GetCurrentThreadId();
	printf("Thread Id = 0x%08x Before InterLocked Count = %d\n",dThreadId,count);
	for (int n = 0; n < (int)arglp; n++)
	{
		InterlockedIncrement(&count); // Interlocked Add Func
	}
	printf("Thread Id = 0x%08x After InterLocked Count = %d\n", dThreadId, count);
	return 0;
}

DWORD WINAPI NoInterLockedChildThreadProcedure(LPVOID arglp)
{
	DWORD dThreadId = GetCurrentThreadId();
	printf("Thread Id = 0x%08x Before Nothing Count = %d\n", dThreadId, count);
	for (int n = 0; n < (int)arglp; n++)
	{
		count++;
	}
	printf("Thread Id = 0x%08x After Nothing Count = %d\n", dThreadId, count);
	return 0;
	
}

int CreateThreadFunc(int argv)
{
	HANDLE hChildThread[3];
	DWORD dUnusedThreadId;

	int check = argv;

	printf("Input Loop Num : ");
	scanf_s("%d", &iloopnum);
	
	if (check == 1)
	{
		hChildThread[0] = (HANDLE)CreateThread(NULL, 0, NoInterLockedChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[1] = (HANDLE)CreateThread(NULL, 0, NoInterLockedChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[2] = (HANDLE)CreateThread(NULL, 0, NoInterLockedChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
	}
	else if (check == 2)
	{
		hChildThread[0] = (HANDLE)CreateThread(NULL, 0, InterLockedChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[1] = (HANDLE)CreateThread(NULL, 0, InterLockedChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[2] = (HANDLE)CreateThread(NULL, 0, InterLockedChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
	}
	else
	{
		printf("check = %d Is illegal Value\n",check);
		return 0;
	}

	if (!hChildThread[0] || !hChildThread[1] || !hChildThread[2])
	{
		printf("Error CreateChildThread Error");
		return NULL;
	}

	DWORD dwResult = WaitForMultipleObjects(3, hChildThread, TRUE, INFINITE);

	if (dwResult != WAIT_OBJECT_0)
	{
		printf("Error WaitForMultipleObject()");
		return 0;
	}

	CloseHandle(hChildThread[0]);
	CloseHandle(hChildThread[1]);
	CloseHandle(hChildThread[2]);
	return 0;

}

int main(void)
{
	int i = 0;
	while (1)
	{
		printf("1. Nothing\n2. Set InterLocked\nInput Number : ");
		scanf_s("%d", &i);
		if (i == 1 || i == 2)
		{
			CreateThreadFunc(i);
			return 0;
		}
		else
		{
			printf("Input legal Value\n");
		}
	}


}
```

### [#] 실행 결과
실행 결과는 아래와 같습니다.
CriticalSection과 달리 변수 연산만 임계 영역에 존재하기 때문에 10000000단위로 끝나지 않습니다.

```
1. Nothing
2. Set InterLocked
Input Number : 1
Input Loop Num : 10000000
Thread Id = 0x00004fb8 Before Nothing Count = 0
Thread Id = 0x000063d0 Before Nothing Count = 5865
Thread Id = 0x00006364 Before Nothing Count = 54040
Thread Id = 0x00006364 After Nothing Count = 8021225
Thread Id = 0x000063d0 After Nothing Count = 8974293
Thread Id = 0x00004fb8 After Nothing Count = 10627299

1. Nothing
2. Set InterLocked
Input Number L: 2
Input Loop Num : 10000000
Thread Id = 0x00004fc0 Before InterLocked Count = 0
Thread Id = 0x0000409c Before InterLocked Count = 0
Thread Id = 0x00005ab4 Before InterLocked Count = 21530 // Thread 생성 시 이미 임계 영역에서 count++이 진행중입니다.
Thread Id = 0x0000409c After InterLocked Count = 21388029 // 변수 연산만 InterLocked 함수를 이용하기 때문에 결과값이 10000000단위로 나오지 않습니다.
Thread Id = 0x00004fc0 After InterLocked Count = 29615583
Thread Id = 0x00005ab4 After InterLocked Count = 30000000
```

커널 레벨 동기화는 다음 포스팅에서 정리하겠습니다.


# #Relation Post
1. [Windows Synchronization_0x01](https://holi4m.github.io/windows/2020/03/12/Synchronization-1/)

# #Reference
1. [MSDN](https://docs.microsoft.com/ko-kr/windows/win32/sync/synchronization)
2. [뇌를 자극하는 윈도우즈 시스템 프로그래밍](http://www.hanbit.co.kr/store/books/look.php?p_code=B7673779595)
3. [Blog 1](https://cesl.tistory.com/entry/Mutex%EC%99%80-Semaphore-%EA%B5%AC%ED%98%84%ED%95%98%EC%97%AC-%EB%B9%84%EA%B5%90)
4. [Blog 2](https://5kyc1ad.tistory.com/252)


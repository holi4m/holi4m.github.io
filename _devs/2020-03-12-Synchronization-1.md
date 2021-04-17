---
layout: post
title: Windows Synchronization_0x01
tags:
- Windows
---
# # Introduce
Windows Synchronization_0x00 포스팅에 이어서 커널 레벨의 윈도우 동기화를 알아보겠습니다.


## ## Kernel Level Synchronization
- Kernel Level
	- Mutex
	- Semaphore
	- Event
	- Etc...

커널 레벨에서 이루어지는 동기화는 Create 계열 함수를 이용하여 오브젝트를 만드는 경우 프로세스 간 공유가 가능한 장점이 있습니다. 
>CreateMutex, CreateSemaphore, CreateEvent, CreateWaitableTimer  

하지만 유저 레벨에 비해서 속도가 느린 단점이 있습니다.

해당 포스팅에서는 Kernel Level의 Mutex, Semaphore, Event를 구현하여 Windows Synchronization을 테스트 하였습니다.

### [#] Mutex.cpp
Mutex 함수를 적용한 소스입니다.
유저 레벨 함수와 성능 차이가 궁금할 경우 count++ 부분에만 WaitForSingleObject와 Mutex를 적용하면 됩니다.

```cpp
#include <stdio.h>
#include <process.h>
#include <Windows.h>

HANDLE hMutex;
int count;
int iloopnum = 0;

DWORD WINAPI MutexChildThreadProcedure(LPVOID arglp)
{
	DWORD dThreadId = GetCurrentThreadId();
	DWORD dwResult = WaitForSingleObject(hMutex, INFINITE); // Waiting Signaled 

	printf("Thread Id = 0x%08x Before Mutex Count = %d\n", dThreadId, count);

	for (int n = 0; n < (int)arglp; n++)
	{	

		if (dwResult == WAIT_OBJECT_0) // WaitForSingleObject Check
		{
			count++;
		}
		else
		{
			printf("Error calling WaitForSingleObject()\n");
			return -1;
		}
	}

	printf("Thread Id = 0x%08x After Mutex Count = %d\n", dThreadId, count);

	ReleaseMutex(hMutex);

	return 0;
}

DWORD WINAPI NoMutexChildThreadProcedure(LPVOID arglp)
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

	hMutex = CreateMutex(NULL, FALSE, NULL); // Create mutex Object

	printf("Input Loop Num : ");
	scanf_s("%d", &iloopnum);

	while (1)
	{
		if (check == 1)
		{
			hChildThread[0] = (HANDLE)CreateThread(NULL, 0, NoMutexChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
			hChildThread[1] = (HANDLE)CreateThread(NULL, 0, NoMutexChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
			hChildThread[2] = (HANDLE)CreateThread(NULL, 0, NoMutexChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
			break;
		}
		else if (check == 2)
		{
			hChildThread[0] = (HANDLE)CreateThread(NULL, 0, MutexChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
			hChildThread[1] = (HANDLE)CreateThread(NULL, 0, MutexChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
			hChildThread[2] = (HANDLE)CreateThread(NULL, 0, MutexChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
			break;
		}
		else
		{
			printf("check = %d Is illegal Value\n", check);
		}
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
	CloseHandle(hMutex);

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
대부분의 스레드 코드가 단일 스레드만 접근이 가능하여 10000000단위로 쓰레드가 종료됩니다.  

```
1. Nothing
2. Set Mutex
Input Number : 1
Input Loop Num : 10000000
Thread Id = 0x000011d0 Before Nothing Count = 0
Thread Id = 0x00003c54 Before Nothing Count = 0
Thread Id = 0x00002508 Before Nothing Count = 220182
Thread Id = 0x00003c54 After Nothing Count = 5503605
Thread Id = 0x000011d0 After Nothing Count = 8950656
Thread Id = 0x00002508 After Nothing Count = 12777291

1. Nothing
2. Set Mutex
Input Number : 2
Input Loop Num : 10000000
Thread Id = 0x00000ab0 Before Mutex Count = 0
Thread Id = 0x00000ab0 After Mutex Count = 10000000
Thread Id = 0x0000057c Before Mutex Count = 10000000
Thread Id = 0x0000057c After Mutex Count = 20000000
Thread Id = 0x0000360c Before Mutex Count = 20000000
Thread Id = 0x0000360c After Mutex Count = 30000000
```

### [#] Event.cpp
Event 함수를 적용한 소스입니다.  
Event는 동기화 보다는 함수 호출 유무, 로직 분기에 더 많이 쓰이는 편입니다.  
CreateEvent에서 2번째 Argument를 True로 할 경우 Manuel-Reset을 하는 동안 스레드가 같이 실행될 수 있으므로  
동기화 용도로 사용하실 때는 Auto-Reset을 추천드립니다.  

```cpp
#include <stdio.h>
#include <process.h>
#include <Windows.h>

HANDLE hEvent;
int count;
int iloopnum = 0;

DWORD WINAPI EventChildThreadProcedure(LPVOID arglp)
{

	DWORD dwResult = WaitForSingleObject(hEvent, INFINITE); // Waiting Signaled And Auto-Reset Set
	DWORD dThreadId = GetCurrentThreadId();

	printf("Thread Id = 0x%08x Before Event Count = %d\n", dThreadId, count);

	if (dwResult == WAIT_OBJECT_0) // WaitForSingleObject Check
	{
		for (int n = 0; n < (int)arglp; n++)
		{
			count++;			
		}
	}

	else if (WAIT_FAILED)
	{
		printf("Error calling WaitForSingleObject()\n");
		return -1;
	}

	printf("Thread Id = 0x%08x After Event Count = %d\n", dThreadId, count);
	
	SetEvent(hEvent); // Set Signaled 

	return 0;
}

DWORD WINAPI NoEventChildThreadProcedure(LPVOID arglp)
{
	DWORD dThreadId = GetCurrentThreadId();

	printf("Thread Id = 0x%08x Before Event Count = %d\n", dThreadId, count);

	for (int n = 0; n < (int)arglp; n++)
	{
		count++;
	}

	printf("Thread Id = 0x%08x After Event Count = %d\n", dThreadId, count);

	return 0;
}

int EventFunc(void)
{
	HANDLE hChildThread[3];
	DWORD dUnusedThreadId;
	LPCSTR lpcEventName = "nEvent";

	hEvent = CreateEvent(NULL, FALSE , TRUE , lpcEventName); // No Duplicate, Auto Reset, Defualt Signal, Name

	printf("Input Loop Num : ");
	scanf_s("%d", &iloopnum);
	
	hChildThread[0] = (HANDLE)CreateThread(NULL, 0, EventChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
	hChildThread[1] = (HANDLE)CreateThread(NULL, 0, EventChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
	hChildThread[2] = (HANDLE)CreateThread(NULL, 0, EventChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
	
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
	CloseHandle(hEvent);
	
	return 0;
}

int NonEventFunc(void)
{

	HANDLE hChildThread[3];
	DWORD dUnusedThreadId;

	printf("Input Loop Num : ");
	scanf_s("%d", &iloopnum);

	hChildThread[0] = (HANDLE)CreateThread(NULL, 0, NoEventChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
	hChildThread[1] = (HANDLE)CreateThread(NULL, 0, NoEventChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
	hChildThread[2] = (HANDLE)CreateThread(NULL, 0, NoEventChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);

	if (!hChildThread[0] || !hChildThread[1] || !hChildThread[2])
	{
		printf("Error CreateChildThread Error");
		return NULL;
	}

	DWORD dwResult = WaitForMultipleObjects(3, hChildThread, TRUE, INFINITE);

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
		printf("1. Nothing\n2. Set Event\nInput Number = ");
		scanf_s("%d", &i);
		if (i == 1)
		{
			NonEventFunc();
			return 0;
		}
		else if (i == 2)
		{
			EventFunc();
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
실행 결과는 Mutex와 동일합니다.  

```
1. Nothing
2. Set Event
Input Number = 1
Input Loop Num : 10000000
Thread Id = 0x00003564 Before Nothing Count = 0
Thread Id = 0x00005f24 Before Nothing Count = 5007
Thread Id = 0x00005788 Before Nothing Count = 700821
Thread Id = 0x00005f24 After Nothing Count = 6823708
Thread Id = 0x00003564 After Nothing Count = 11919642
Thread Id = 0x00005788 After Nothing Count = 12130188

1. Nothing
2. Set Event
Input Number = 2
Input Loop Num : 10000000
Thread Id = 0x00005d8c Before Event Count = 0
Thread Id = 0x00005d8c After Event Count = 10000000
Thread Id = 0x00001684 Before Event Count = 10000000
Thread Id = 0x00001684 After Event Count = 20000000
Thread Id = 0x00005804 Before Event Count = 20000000
Thread Id = 0x00005804 After Event Count = 30000000
```

### [#] Semaphore.cpp
Semaphore 함수를 적용한 소스입니다.  
Mutex가 1인용이라면 Semaphore는 다인용입니다.  
해당 소스에서는 공유리소스를 하나만 사용하여 Mutex와 동일하게 실행됩니다.  

```cpp
#include <stdio.h>
#include <Windows.h>
#include <process.h>

HANDLE hSemaphore;
int count;
int iloopnum = 0;
LONG lSemaphoreCount = 3;
LPLONG lpPreviousCount = &lSemaphoreCount;

DWORD WINAPI NoSemaphoreChildThreadProcedure(LPVOID arglp)
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

DWORD WINAPI SemaphoreChildThreadProcedure(LPVOID arglp)
{
	DWORD dThreadId = GetCurrentThreadId();
	printf("Before Semaphore PreviousCount= %d\n", *lpPreviousCount);
	DWORD dwResult = WaitForSingleObject(hSemaphore, INFINITE); // Waiting Signaled 
	printf("Thread Id = 0x%08x Before Semaphore Count = %d\n", dThreadId, count);

	for (int n = 0; n < (int)arglp; n++)
	{

		if (dwResult == WAIT_OBJECT_0) // WaitForSingleObject Check
		{
			count++;
		}
		else
		{
			printf("Error calling WaitForSingleObject()\n");
			return -1;
		}
	}

	printf("Thread Id = 0x%08x After Semaphore Count = %d\n", dThreadId, count);
	ReleaseSemaphore(hSemaphore,1,lpPreviousCount);
	printf("After Semaphore PreviousCount= %d\n", *lpPreviousCount);

	return 0;
}

int CreateThreadFunc(int argv)
{
	HANDLE hChildThread[3];
	DWORD dUnusedThreadId;

	int check = argv;

	hSemaphore = CreateSemaphore(NULL, lSemaphoreCount, 5 ,NULL); // Create Semaphore Object

	printf("Input Loop Num : ");
	scanf_s("%d", &iloopnum);

	if (check == 1)
	{
		hChildThread[0] = (HANDLE)CreateThread(NULL, 0, NoSemaphoreChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[1] = (HANDLE)CreateThread(NULL, 0, NoSemaphoreChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[2] = (HANDLE)CreateThread(NULL, 0, NoSemaphoreChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
	}
	else if (check == 2)
	{
		hChildThread[0] = (HANDLE)CreateThread(NULL, 0, SemaphoreChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[1] = (HANDLE)CreateThread(NULL, 0, SemaphoreChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
		hChildThread[2] = (HANDLE)CreateThread(NULL, 0, SemaphoreChildThreadProcedure, (LPVOID)iloopnum, 0, &dUnusedThreadId);
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
	CloseHandle(hSemaphore);

	return 0;
}

int main(void)
{
	int i = 0;
	while (1)
	{
		printf("1. Nothing\n2. Set Semaphore\nInput Number : ");
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
실행 결과는 Mutex와 동일합니다.  
Semaphore 카운팅을 위해 Semaphore PreviousCount를 추가하였습니다.  

```
1. Nothing
2. Set Semaphore
Input Number : 1
Input Loop Num : 10000000
Thread Id = 0x00005eb8 Before Nothing Count = 0
Thread Id = 0x000060cc Before Nothing Count = 12737
Thread Id = 0x00001450 Before Nothing Count = 66626
Thread Id = 0x000060cc After Nothing Count = 7644709
Thread Id = 0x00005eb8 After Nothing Count = 10635012
Thread Id = 0x00001450 After Nothing Count = 12379750

1. Nothing
2. Set Semaphore
Input Number : 2
Input Loop Num : 10000000
Before Semaphore PreviousCount= 1
Thread Id = 0x00006084 Before Semaphore Count = 0
Before Semaphore PreviousCount= 1
Before Semaphore PreviousCount= 1
Thread Id = 0x00006084 After Semaphore Count = 10000000
After Semaphore PreviousCount= 0
Thread Id = 0x00005d68 Before Semaphore Count = 10000000
Thread Id = 0x00005d68 After Semaphore Count = 20000000
After Semaphore PreviousCount= 0
Thread Id = 0x000061c0 Before Semaphore Count = 20000000
Thread Id = 0x000061c0 After Semaphore Count = 30000000
After Semaphore PreviousCount= 0
```

# #Reference
1. [MSDN](https://docs.microsoft.com/ko-kr/windows/win32/sync/synchronization)
2. [뇌를 자극하는 윈도우즈 시스템 프로그래밍](http://www.hanbit.co.kr/store/books/look.php?p_code=B7673779595)
3. [Blog 1](https://cesl.tistory.com/entry/Mutex%EC%99%80-Semaphore-%EA%B5%AC%ED%98%84%ED%95%98%EC%97%AC-%EB%B9%84%EA%B5%90)
4. [Blog 2](https://5kyc1ad.tistory.com/252)


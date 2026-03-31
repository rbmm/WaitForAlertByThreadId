# WaitForAlertByThreadId

```
NTSYSCALLAPI
NTSTATUS
NTAPI
ZwAlertThreadByThreadId(
    _In_ HANDLE ThreadId
    );
```
    
return values:
`STATUS_SUCCESS`
`STATUS_INVALID_CID` - ThreadId not valid
`STATUS_ACCESS_DENIED` - ThreadId to thread from another process

so `ZwAlertThreadByThreadId` can be used to alert thread only from current process.
`ZwAlertThreadByThreadId` make effect only on thread, which wait in `ZwWaitForAlertByThreadId`
it have no effect on usual, alertable wait ( `ZwDelayExecution`, `ZwWaitForSingleObject`, etc)

```
NTSYSCALLAPI
NTSTATUS
NTAPI
ZwWaitForAlertByThreadId(
    _In_ PVOID Address,
    _In_opt_ PLARGE_INTEGER Timeout
    );
```    
Address - can be any value. not interpreted by system and not returned back. can be used only for debugging.
usually here can be pass 0 or address of SRWLOCK or CRITICAL_SECTION

return value - `STATUS_ALERTED` or `STATUS_TIMEOUT`

when ZwAlertThreadByThreadId called - it lock are target thread wait with wait reason WrAlertByThreadId - if yes - unwait thread.
if no, set flag in thread object. several calls to ZwAlertThreadByThreadId have the same effect as single call (set flag)
```
		ZwAlertThreadByThreadId((HANDLE)(ULONG_PTR)GetCurrentThreadId());
		ZwAlertThreadByThreadId((HANDLE)(ULONG_PTR)GetCurrentThreadId());
		LARGE_INTEGER li = {};
		if (STATUS_ALERTED != ZwWaitForAlertByThreadId(0, &li))__debugbreak();
		if (STATUS_TIMEOUT != ZwWaitForAlertByThreadId(0, &li))__debugbreak();
```		
say typical usage of event
```
struct {
  HANDLE m_hEvent = CreateEventW(0, TRUE, 0, 0);
  
  void fn(){
    //...
    WaitForSingleObject(m_hEvent, INFINITE);
    //...
  }
  
  void gn(){
    //...
    SetEvent(m_hEvent);
    //...
  }
};
```
can be (for first look) replaced to
```
struct {
  HANDLE m_dwThreadId = (HANDLE)(ULONG_PTR)GetCurrentThreadId();
  
  void fn(){ // called only from m_dwThreadId
    //...
    ZwWaitForAlertByThreadId(0, 0);
    //...
  }
  
  void gn(){ // on arbitrary thread
    //...
    ZwAlertThreadByThreadId(m_dwThreadId);
    //...
  }
};
```
advantage - we not need create event object, handle error on it create, close it.
but..
at first m_dwThreadId must be valid at time of call ZwAlertThreadByThreadId (so thread m_dwThreadId must not terminated before ZwAlertThreadByThreadId)
thread m_dwThreadId must not try enter to srw lock or crit sec, after ZwAlertThreadByThreadId and before our ZwWaitForAlertByThreadId
otherwise our alert can be lost or code can enter to infinite loop.

demo example
```
static ULONG WINAPI LoopThread(PVOID pcs)
{
	ZwAlertThreadByThreadId((HANDLE)(ULONG_PTR)reinterpret_cast<LPCRITICAL_SECTION>(pcs)->OwningThread);
	ZwAlertThreadByThreadId((HANDLE)(ULONG_PTR)GetCurrentThreadId()); // !!! <-- this
	EnterCriticalSection((LPCRITICAL_SECTION)pcs);
	LeaveCriticalSection((LPCRITICAL_SECTION)pcs);
	return 0;
}

		CRITICAL_SECTION cs;
		InitializeCriticalSection(&cs);
		EnterCriticalSection(&cs);
		
		if (HANDLE hThread = CreateThread(0, 0, LoopThread, &cs, 0, 0))
		{
			ZwWaitForAlertByThreadId(0, 0);
			Sleep(1000);// give thread time to try enter crit sec
			LeaveCriticalSection(&cs);
			WaitForSingleObject(hThread, INFINITE);
			NtClose(hThread);
		}
		else
		{
			LeaveCriticalSection(&cs);
		}
		DeleteCriticalSection(&cs);
```		
as result LoopThread enter to infinite loop 
```	
ntdll.dll!RtlpWaitOnCriticalSection + 2bb
ntdll.dll!RtlpEnterCriticalSectionContended + 1ef
ntdll.dll!RtlEnterCriticalSection + f2
PoC.exe!NT::LoopThread + 33
kernel32.dll!BaseThreadInitThunk + 17
ntdll.dll!RtlUserThreadStart + 2c

@@0:
    mov         rdx,rbx
    mov         rbx,qword ptr [rbx+10h] ; rbx == [rbx+10h]
    mov         qword ptr [rbx+18h],rdx
    cmp         qword ptr [rbx+20h],0
    jne         @@1
    jmp         @@0
```    
api very frequently use srw or crit sec. and heap - allocate and free heap use locks. most api use heap alloc/free. 

so for correct usage need call ZwAlertThreadByThreadId only when we exactly know that target thread wait on alert by id or just begin wait, without any additional api calls
also need to be shure that after thread stop wait in ZwWaitForAlertByThreadId, no pendig alert flag is set in thread

so solution, can be next:
```
union TIDFA {
	LONG m_dwThreadId;
	struct {
		ULONG m_wait : 1;
		ULONG m_alert : 1;
	};

	TIDFA(ULONG dwThreadId = GetCurrentThreadId()) : m_dwThreadId(dwThreadId)
	{
	}

	void Alert()
	{
		union {
			ULONG dwThreadId;
			struct {
				ULONG wait : 1;
				ULONG alert : 1;
			};
		};
		union {
			ULONG _dwThreadId;
			struct {
				ULONG _wait : 1;
				ULONG _alert : 1;
			};
		};

		dwThreadId = m_dwThreadId;

		for (;;)
		{
			_dwThreadId = dwThreadId;

			alert = 1, wait = 0;

			if (_dwThreadId == (dwThreadId = InterlockedCompareExchange(&m_dwThreadId, dwThreadId, _dwThreadId)))//--0
			{
				if (wait && !alert)
				{
					wait = 0;
					if (0 > ZwAlertThreadByThreadId((HANDLE)(ULONG_PTR)dwThreadId))
					{
						__debugbreak();
					}
				}

				break;
			}
		}
	}

	NTSTATUS WaitForAlert(_In_opt_ PLARGE_INTEGER Timeout = 0)
	{
		union {
			ULONG dwThreadId;
			struct {
				ULONG wait : 1;
				ULONG alert : 1;
			};
		};
		union {
			ULONG _dwThreadId;
			struct {
				ULONG _wait : 1;
				ULONG _alert : 1;
			};
		};

		dwThreadId = m_dwThreadId;

		for(;;)
		{
			_dwThreadId = dwThreadId;

			if (wait)
			{
				__debugbreak();
			}

			if (alert)
			{
				alert = 0;
				wait = 0;
			}
			else
			{
				wait = 1;
			}

			BOOL bNeedWait = wait;

			if (_dwThreadId == (dwThreadId = InterlockedCompareExchange(&m_dwThreadId, dwThreadId, _dwThreadId)))//--1
			{			
				if (bNeedWait)
				{
					NTSTATUS status = ZwWaitForAlertByThreadId(this, Timeout);

					dwThreadId = m_dwThreadId;

					for (;;)
					{
						_dwThreadId = dwThreadId;

						wait = 0, alert = 0;
						
						if (_dwThreadId == (dwThreadId = InterlockedCompareExchange(&m_dwThreadId, dwThreadId, _dwThreadId)))//--2
						{
							if (alert)
							{
								if (STATUS_ALERTED != status)
								{
									LARGE_INTEGER zt = {};
									
									ULONG spinCount = 0;
									while (STATUS_ALERTED != ZwWaitForAlertByThreadId(this, &zt))
									{
										switch (++spinCount >> 3)
										{
										case 0: YieldProcessor();
											continue;
										case 1: SwitchToThread();
											continue;
										}

										Sleep(1);
									}

									return STATUS_ALERTED;
								}
							}

							return status;
						}
					}
				}

				return STATUS_ALERTED;
			}
		} 
	}
};
```

so correct code is

```
struct {
  TIDFA m_tid;
  
  void fn(){ // called only from m_dwThreadId
    //...
    m_tid.WaitForAlert();
    //...
  }
  
  void gn(){ // on arbitrary thread
    //...
    m_tid.Alert();
    //...
  }
};
```

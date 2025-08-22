1. Repeat the walk-through by yourself. Draw the stack layout, including
parameters and local variables.

Walkthrough (drawings are on paper)
<pre>
01: ; BOOL __stdcall DllMain(HINSTANCE hinstDLL, DWORD fdwReason,
 ; LPVOID lpvReserved)
02: _DllMain@12 proc near
03: 55 push ebp                                                         save ebp
04: 8B EC mov ebp, esp                                                  move esp to ebp
05: 81 EC 30 01 00+ sub esp, 130h                                       make 130 bytes of stack space
06: 57 push edi                                                         save edi
07: 0F 01 4D F8 sidt fword ptr [ebp-8]                                  save interrupt table location to ebp-8 (8 byte memory, 6 byte for content (high 4 base addr, low 2 limit))
08: 8B 45 FA mov eax, [ebp-6]						read base addr to eax (32 bit register)

09: 3D 00 F4 03 80 cmp eax, 8003F400h                                   
10: 76 10 jbe short loc_10001C88 (line 18)
11: 3D 00 74 04 80 cmp eax, 80047400h
12: 73 09 jnb short loc_10001C88 (line 18)                              limit check eax

13: 33 C0 xor eax, eax                                                  
14: 5F pop edi                                                          
15: 8B E5 mov esp, ebp                                                  
16: 5D pop ebp
17: C2 0C 00 retn 0Ch                                                   cleanup and return

18: loc_10001C88:
19: 33 C0 xor eax, eax                                                  zero eax
20: B9 49 00 00 00 mov ecx, 49h                                         set ecx to 0x49
21: 8D BD D4 FE FF+ lea edi, [ebp-12Ch]                                 set edi to address of ebp-12c (0x130 - 4 bytes)
22: C7 85 D0 FE FF+ mov dword ptr [ebp-130h], 0                         zero last byte of stack var
23: 50 push eax                                                         push 0
24: 6A 02 push 2                                                        push 2
25: F3 AB rep stosd                                                     zero ecx (49h) * 4 bytes == 0x124
26: E8 2D 2F 00 00 call CreateToolhelp32Snapshot                        call CreateToolhelp32Snapshot(2, 0) -> returns a snapshot of all processes, prepared for Processfirst
27: 8B F8 mov edi, eax                                                  set edi to the handle of the process snapshot

28: 83 FF FF cmp edi, 0FFFFFFFFh                                        
29: 75 09 jnz short loc_10001CB9 (line 35)                              check if handle is valid

30: 33 C0 xor eax, eax
31: 5F pop edi
32: 8B E5 mov esp, ebp
33: 5D pop ebp
34: C2 0C 00 retn 0Ch                                                   cleanup and return

35: loc_10001CB9:
36: 8D 85 D0 FE FF+ lea eax, [ebp-130h]                                 set eax to ebp-130h
37: 56 push esi                                                         save esi
38: 50 push eax                                                         save eax
39: 57 push edi                                                         save edi
40: C7 85 D0 FE FF+ mov dword ptr [ebp-130h], 128h                      set ebp-130h to 0x128
41: E8 FF 2E 00 00 call Process32First                                  call Process32First(edi, ebp-130h) -> handle from snapshot, local variable containing space for struct on stack
42: 85 C0 test eax, eax                                                 
43: 74 4F jz short loc_10001D24 (line 70)                               checks if last process was loaded into ebp-130h
44: 8B 35 C0 50 00+ mov esi, ds:_stricmp                                sets esi to address of str_comp
45: 8D 8D F4 FE FF+ lea ecx, [ebp-10Ch]                                 set ecx to address ebp-10Ch (as ebp-130h is byte 0 of the struct, ebp-10c is byte 0x24 of the struct)

"
typedef struct tagPROCESSENTRY32 {
0x00  DWORD     dwSize;
0x04  DWORD     cntUsage;
0x08  DWORD     th32ProcessID;
0x0c  ULONG_PTR th32DefaultHeapID;
0x10  DWORD     th32ModuleID;
0x14  DWORD     cntThreads;
0x18  DWORD     th32ParentProcessID;
0x1c  LONG      pcPriClassBase; (LONG is 32bit)
0x20  DWORD     dwFlags;
0x24  CHAR      szExeFile[MAX_PATH];
} PROCESSENTRY32;
"

46: 68 50 7C 00 10 push 10007C50h                                       
47: 51 push ecx
48: FF D6 call esi ; _stricmp                                           call _stricmp(szExeFile, "explorer.exe")
49: 83 C4 08 add esp, 8                                                 clear args from stack
50: 85 C0 test eax, eax
51: 74 26 jz short loc_10001D16 (line 66)                               jump if strings are equal
52: loc_10001CF0:
53: 8D 95 D0 FE FF+ lea edx, [ebp-130h]                                 edx = ebp-130h
54: 52 push edx                                                         
55: 57 push edi
56: E8 CD 2E 00 00 call Process32Next                                   call Process32Next(edi, edx) -> handle from snapshot, local variable containing space for struct on stack
57: 85 C0 test eax, eax                                                 test if done
58: 74 23 jz short loc_10001D24 (line 70)                               jump to 70 if done
59: 8D 85 F4 FE FF+ lea eax, [ebp-10Ch]
60: 68 50 7C 00 10 push 10007C50h
61: 50 push eax
62: FF D6 call esi ; _stricmp                                           call _stricmp(szExeFile, "explorer.exe")
63: 83 C4 08 add esp, 8                                                 cleanup
64: 85 C0 test eax, eax                                                 
65: 75 DA jnz short loc_10001CF0 (line 52)                              jump back if strings are unequal
66: loc_10001D16:
67: 8B 85 E8 FE FF+ mov eax, [ebp-118h]                                 eax = th32ParentProcessID;
68: 8B 8D D8 FE FF+ mov ecx, [ebp-128h]                                 ecx = th32ProcessID;
69: EB 06 jmp short loc_10001D2A (line 73)                              jump to 73
70: loc_10001D24:
71: 8B 45 0C mov eax, [ebp+0Ch]                                         eax = fdwReason
72: 8B 4D 0C mov ecx, [ebp+0Ch]                                         ecx = fdwReason
73: loc_10001D2A:
74: 3B C1 cmp eax, ecx                                                  
75: 5E pop esi                                                          restore esi
76: 75 09 jnz short loc_10001D38 (line 82)                              jump to 82 if eax != ecx
77: 33 C0 xor eax, eax
78: 5F pop edi
79: 8B E5 mov esp, ebp
80: 5D pop ebp
81: C2 0C 00 retn 0Ch                                                   cleanup and return
82: loc_10001D38:
83: 8B 45 0C mov eax, [ebp+0Ch]                                         eax = fdwReason
84: 48 dec eax                                                          eax--
85: 75 15 jnz short loc_10001D53 (line 93)                              jump if fdwreason was not DLL_PROCESS_ATTACH
86: 6A 00 push 0
87: 6A 00 push 0
88: 6A 00 push 0
89: 68 D0 32 00 10 push 100032D0h
90: 6A 00 push 0
91: 6A 00 push 0
92: FF 15 20 50 00+ call ds:CreateThread                                call CreateThread(0, 0, 100032D0h, 0, 0, 0)
93: loc_10001D53:
94: B8 01 00 00 00 mov eax, 1                                           
95: 5F pop edi
96: 8B E5 mov esp, ebp
97: 5D pop ebp
98: C2 0C 00 retn 0Ch                                                   cleanup and return 1
99: _DllMain@12 endp
</pre>

2. In the example walk-through, we did a nearly one-to-one translation of
the assembly code to C. As an exercise, re-decompile this whole function
so that it looks more natural. What can you say about the developer’s skill
level/experience? Explain your reasons. Can you do a better job?
 <pre>
typedef struct _IDTR {
  DWORD base;
  SHORT limit;
} IDTR, *PIDTR;

BOOL __stdcall DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved) {
  IDTR idtr;
  Handle h;
  PROCESSENTRY32 entry;
  DWORD parentID, procID;
  __sidt(&idtr);
  if (idtr.base <= 0x8003F400 || idtr.base > 0x80047400) {
    memset(&entry, 0, sizeof(PROCESSENTRY32)): // assumption
    h = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (h != INVALID_HANDLE_VALUE) {
      entry.dwSize = 0x128;
      parentID = procID = fdwReason;
      if (Process32First(h, &entry)) {
        do {
          if (stricmp(procentry.szExeFile, "explorer.exe")) {
            parentID = procentry.th32ParentProcessID;
            procID = procentry.th32ProcessID;
            break;
          }
        } while(Process32Next(h, &entry));
      }
      if (parentID != procID) {
        if (fdwReason != DLL_PROCESS_ATTACH) {
          CreateThread(0, 0, 0x100032D0, 0, 0, 0);
        }
        return TRUE;
      }
    }
  }
  return FALSE;
}
</pre>

The developer seems to have a medium skill level, high enough to produce well working code, low enough that they fall for problems
problems: 
idtr checking flawed
uses parentID and processID to pass information about finding explorer instead of just Creating the Thread there and then

 <pre>
BOOL __stdcall DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved) {
  Handle h;
  PROCESSENTRY32 entry;
  memset(&entry, 0, sizeof(PROCESSENTRY32)): // assumption
  h = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
  if (h != INVALID_HANDLE_VALUE) {
    entry.dwSize = 0x128;
    if (Process32First(h, &entry)) {
      do {
        if (stricmp(procentry.szExeFile, "explorer.exe")) {
          if (fdwReason != DLL_PROCESS_ATTACH) {
            CreateThread(0, 0, 0x100032D0, 0, 0, 0);
          }
          return TRUE;
        }
      } while(Process32Next(h, &entry));
    }
  }
  return FALSE;
}
</pre>
Reasoning: got rid of IDTR checking, made thread creation trigger asap

3. In some of the assembly listings, the function name has a @ prefi x followed
by a number. Explain when and why this decoration exists.
The @ prefix lists the location relative to the compilation unit

4. Implement the following functions in x86 assembly: strlen, strchr, memcpy, memset, strcmp, strset.

strlen:
<pre>
.globl strlen
.type strlen, @function
strlen:
    push ebp
    mov ebp, esp
    push edi
    mov edi, [ebp + 8]
    xor eax, eax
    mov ecx, -1
    repne scasb
    not ecx
    dec ecx
    mov eax, ecx
    pop edi
    pop ebp
    ret
</pre>

strchr:
<pre>
 .globl strchr
.type strchr, @function
strchr:
    push ebp
    mov ebp, esp
    push edi
    mov edi, [ebp + 8]
    xor eax, eax
    mov ecx, -1
    repne scasb
    not ecx
    dec ecx
    mov edi, [ebp + 8]
    mov eax, [ebp + 12]
    repne scasb
    je found
    mov edi, 1
found:
    dec edi
    mov eax, edi
    pop edi
    pop ebp
    ret
</pre>

memcpy:
<pre>
 .globl memcpy
.type memcpy, @function
memcpy:
    push ebp
    mov ebp, esp
    push esi
    push edi
    mov ecx, [ebp + 16]
    mov edi, [ebp + 8]
    mov esi, [ebp + 12]
    rep movsb
    pop edi
    pop esi
    mov eax, [ebp + 8]
    pop ebp
    ret
</pre>

memset:
<pre>
 .globl memset
.type memset, @function
memset:
    push ebp
    mov ebp, esp
    push edi
    mov ecx, [ebp + 16]
    mov edi, [ebp + 8]
    mov eax, [ebp + 12]
    rep stosb
    pop edi
    mov eax, [ebp + 8]
    pop ebp
    ret
</pre>

strcmp:
<pre>
 .globl strcmp
.type strcmp, @function
strcmp:
    push ebp
    mov ebp, esp
    push esi
    push edi
    mov esi, [ebp + 12]
    mov edi, [ebp + 8]
    mov ecx, -1
    repe cmpsb
    mov al, [edi - 1]
    sub al, [esi - 1]
    movsx eax, al
    pop edi
    pop esi
    pop ebp
    ret
</pre>

strset:
<pre>
 .globl strset
.type strset, @function
strset:
    push ebp
    mov ebp, esp
    push edi
    mov edi, [ebp + 8]
    xor eax, eax
    mov ecx, -1
    repne scasb
    not ecx
    dec ecx
    mov edi, [ebp + 8]
    mov eax, [ebp + 12]
    rep stosb
    pop edi
    mov eax, [ebp + 8]
    pop ebp
    ret
</pre>

5. Decompile the following kernel routines in Windows:
 ■ KeInitializeDpc
 ■ KeInitializeApc
 ■ ObFastDereferenceObject (and explain its calling convention)
 ■ KeInitializeQueue
 ■ KxWaitForLockChainValid
 ■ KeReadyThread
 ■ KiInitializeTSS
 ■ RtlValidateUnicodeString

KeInitializeDpc:
<pre>
lkd> uf KeInitializeDpc
nt!KeInitializeDpc:
fffff801`50d446c0 33c0            xor     eax,eax
fffff801`50d446c2 c70113010000    mov     dword ptr [rcx],113h
fffff801`50d446c8 48894138        mov     qword ptr [rcx+38h],rax
fffff801`50d446cc 48894110        mov     qword ptr [rcx+10h],rax
fffff801`50d446d0 48895118        mov     qword ptr [rcx+18h],rdx
fffff801`50d446d4 4c894120        mov     qword ptr [rcx+20h],r8
fffff801`50d446d8 c3              ret
</pre>

As the function does not access the stack for reading purposes, the function most likely follows "Microsoft x64 calling convention convention". So now we have to look at the function doc by microsoft to see:

<pre>
 void KeInitializeDpc(
  [out]          __drv_aliasesMem PRKDPC Dpc,
  [in]           PKDEFERRED_ROUTINE      DeferredRoutine,
  [in, optional] __drv_aliasesMem PVOID  DeferredContext
);
</pre>

As the function is only writing to the memory pointed to by rcx, we have to look what Dpc is. Now looking at the information on KDPC:

<pre>
lkd> dt _KDPC
nt!_KDPC
   +0x000 TargetInfoAsUlong : Uint4B
   +0x000 Type             : UChar
   +0x001 Importance       : UChar
   +0x002 Number           : Uint2B
   +0x008 DpcListEntry     : _SINGLE_LIST_ENTRY
   +0x010 ProcessorHistory : Uint8B
   +0x018 DeferredRoutine  : Ptr64     void 
   +0x020 DeferredContext  : Ptr64 Void
   +0x028 SystemArgument1  : Ptr64 Void
   +0x030 SystemArgument2  : Ptr64 Void
   +0x038 DpcData          : Ptr64 Void
</pre>

We can now decompile the function as follows (assuming for now that xor eax, eax zeroes all of rax):

<pre>
  void KeInitializeDpc(
  [out]          __drv_aliasesMem PRKDPC Dpc,
  [in]           PKDEFERRED_ROUTINE      DeferredRoutine,
  [in, optional] __drv_aliasesMem PVOID  DeferredContext
) {
  Dpc->TargetInfoAsUlong = 0x113;
  Dpc->DpcData = NULL;
  Dpc->ProcessorHistory = 0;
  Dpc->DeferredRoutine = DeferredRoutine;
  Dpc->DeferredContext = DeferredContext;
}
</pre>

KeInitializeApc:
<pre>
 lkd> uf KeInitializeApc
nt!KeInitializeApc:
fffff801`50d41e70 c60112          mov     byte ptr [rcx],12h
fffff801`50d41e73 4c8bd1          mov     r10,rcx
fffff801`50d41e76 c6410258        mov     byte ptr [rcx+2],58h
fffff801`50d41e7a 4183f802        cmp     r8d,2
fffff801`50d41e7e 7449            je      nt!KeInitializeApc+0x59 (fffff801`50d41ec9)  Branch

nt!KeInitializeApc+0x10:
fffff801`50d41e80 488b442428      mov     rax,qword ptr [rsp+28h]
fffff801`50d41e85 44884150        mov     byte ptr [rcx+50h],r8b
fffff801`50d41e89 48894128        mov     qword ptr [rcx+28h],rax
fffff801`50d41e8d 48895108        mov     qword ptr [rcx+8],rdx
fffff801`50d41e91 488b542430      mov     rdx,qword ptr [rsp+30h]
fffff801`50d41e96 48895130        mov     qword ptr [rcx+30h],rdx
fffff801`50d41e9a 488bc2          mov     rax,rdx
fffff801`50d41e9d 48f7d8          neg     rax
fffff801`50d41ea0 4c894920        mov     qword ptr [rcx+20h],r9
fffff801`50d41ea4 481bc9          sbb     rcx,rcx
fffff801`50d41ea7 48234c2440      and     rcx,qword ptr [rsp+40h]
fffff801`50d41eac 48f7da          neg     rdx
fffff801`50d41eaf 1ac0            sbb     al,al
fffff801`50d41eb1 22442438        and     al,byte ptr [rsp+38h]
fffff801`50d41eb5 41884251        mov     byte ptr [r10+51h],al
fffff801`50d41eb9 49894a38        mov     qword ptr [r10+38h],rcx
fffff801`50d41ebd 41c6425200      mov     byte ptr [r10+52h],0
fffff801`50d41ec2 41c6420100      mov     byte ptr [r10+1],0
fffff801`50d41ec7 c3              ret

nt!KeInitializeApc+0x59:
fffff801`50d41ec9 448a824a020000  mov     r8b,byte ptr [rdx+24Ah]
fffff801`50d41ed0 ebae            jmp     nt!KeInitializeApc+0x10 (fffff801`50d41e80)  Branch
</pre>

While searching for the function the best to find was this https://dennisbabkin.com/inside_nt_apc/, showing the following function header:

<pre>
 NTKERNELAPI VOID KeInitializeApc (
    IN PRKAPC Apc,
    IN PKTHREAD Thread,
    IN KAPC_ENVIRONMENT Environment,
    IN PKKERNEL_ROUTINE KernelRoutine,
    IN PKRUNDOWN_ROUTINE RundownRoutine OPTIONAL,
    IN PKNORMAL_ROUTINE NormalRoutine OPTIONAL,
    IN KPROCESSOR_MODE ApcMode,
    IN PVOID NormalContext
    );
typedef enum _KAPC_ENVIRONMENT {
    OriginalApcEnvironment,
    AttachedApcEnvironment,
    CurrentApcEnvironment
} KAPC_ENVIRONMENT;
</pre>

One of the most important structures here seems to be PRKAPC and PKTHREAD which are pointers to a KAPC and KTHREAD respectively which seem to be the following:

<pre>
lkd> dt _KAPC
nt!_KAPC
   +0x000 Type             : UChar
   +0x001 AllFlags         : UChar
   +0x001 CallbackDataContext : Pos 0, 1 Bit
   +0x001 Unused           : Pos 1, 7 Bits
   +0x002 Size             : UChar
   +0x003 SpareByte1       : UChar
   +0x004 SpareLong0       : Uint4B
   +0x008 Thread           : Ptr64 _KTHREAD
   +0x010 ApcListEntry     : _LIST_ENTRY
   +0x020 KernelRoutine    : Ptr64     void 
   +0x028 RundownRoutine   : Ptr64     void 
   +0x030 NormalRoutine    : Ptr64     void 
   +0x020 Reserved         : [3] Ptr64 Void
   +0x038 NormalContext    : Ptr64 Void
   +0x040 SystemArgument1  : Ptr64 Void
   +0x048 SystemArgument2  : Ptr64 Void
   +0x050 ApcStateIndex    : Char
   +0x051 ApcMode          : Char
   +0x052 Inserted         : UChar

 lkd> dt _KTHREAD
nt!_KTHREAD
   +0x000 Header           : _DISPATCHER_HEADER
   +0x018 SListFaultAddress : Ptr64 Void
   +0x020 QuantumTarget    : Uint8B
   +0x028 InitialStack     : Ptr64 Void
   +0x030 StackLimit       : Ptr64 Void
   +0x038 StackBase        : Ptr64 Void
   +0x040 ThreadLock       : Uint8B
   +0x048 CycleTime        : Uint8B
   +0x050 CurrentRunTime   : Uint4B
   +0x054 ExpectedRunTime  : Uint4B
   +0x058 KernelStack      : Ptr64 Void
   +0x060 StateSaveArea    : Ptr64 _XSAVE_FORMAT
   +0x068 SchedulingGroup  : Ptr64 _KSCHEDULING_GROUP
   +0x070 WaitRegister     : _KWAIT_STATUS_REGISTER
   +0x071 Running          : UChar
   +0x072 Alerted          : [2] UChar
   +0x074 AutoBoostActive  : Pos 0, 1 Bit
   +0x074 ReadyTransition  : Pos 1, 1 Bit
   +0x074 WaitNext         : Pos 2, 1 Bit
   +0x074 SystemAffinityActive : Pos 3, 1 Bit
   +0x074 Alertable        : Pos 4, 1 Bit
   +0x074 UserStackWalkActive : Pos 5, 1 Bit
   +0x074 ApcInterruptRequest : Pos 6, 1 Bit
   +0x074 QuantumEndMigrate : Pos 7, 1 Bit
   +0x074 UmsDirectedSwitchEnable : Pos 8, 1 Bit
   +0x074 TimerActive      : Pos 9, 1 Bit
   +0x074 SystemThread     : Pos 10, 1 Bit
   +0x074 ProcessDetachActive : Pos 11, 1 Bit
   +0x074 CalloutActive    : Pos 12, 1 Bit
   +0x074 ScbReadyQueue    : Pos 13, 1 Bit
   +0x074 ApcQueueable     : Pos 14, 1 Bit
   +0x074 ReservedStackInUse : Pos 15, 1 Bit
   +0x074 UmsPerformingSyscall : Pos 16, 1 Bit
   +0x074 TimerSuspended   : Pos 17, 1 Bit
   +0x074 SuspendedWaitMode : Pos 18, 1 Bit
   +0x074 SuspendSchedulerApcWait : Pos 19, 1 Bit
   +0x074 CetUserShadowStack : Pos 20, 1 Bit
   +0x074 BypassProcessFreeze : Pos 21, 1 Bit
   +0x074 Reserved         : Pos 22, 10 Bits
   +0x074 MiscFlags        : Int4B
   +0x078 ThreadFlagsSpare : Pos 0, 2 Bits
   +0x078 AutoAlignment    : Pos 2, 1 Bit
   +0x078 DisableBoost     : Pos 3, 1 Bit
   +0x078 AlertedByThreadId : Pos 4, 1 Bit
   +0x078 QuantumDonation  : Pos 5, 1 Bit
   +0x078 EnableStackSwap  : Pos 6, 1 Bit
   +0x078 GuiThread        : Pos 7, 1 Bit
   +0x078 DisableQuantum   : Pos 8, 1 Bit
   +0x078 ChargeOnlySchedulingGroup : Pos 9, 1 Bit
   +0x078 DeferPreemption  : Pos 10, 1 Bit
   +0x078 QueueDeferPreemption : Pos 11, 1 Bit
   +0x078 ForceDeferSchedule : Pos 12, 1 Bit
   +0x078 SharedReadyQueueAffinity : Pos 13, 1 Bit
   +0x078 FreezeCount      : Pos 14, 1 Bit
   +0x078 TerminationApcRequest : Pos 15, 1 Bit
   +0x078 AutoBoostEntriesExhausted : Pos 16, 1 Bit
   +0x078 KernelStackResident : Pos 17, 1 Bit
   +0x078 TerminateRequestReason : Pos 18, 2 Bits
   +0x078 ProcessStackCountDecremented : Pos 20, 1 Bit
   +0x078 RestrictedGuiThread : Pos 21, 1 Bit
   +0x078 VpBackingThread  : Pos 22, 1 Bit
   +0x078 ThreadFlagsSpare2 : Pos 23, 1 Bit
   +0x078 EtwStackTraceApcInserted : Pos 24, 8 Bits
   +0x078 ThreadFlags      : Int4B
   +0x07c Tag              : UChar
   +0x07d SystemHeteroCpuPolicy : UChar
   +0x07e UserHeteroCpuPolicy : Pos 0, 7 Bits
   +0x07e ExplicitSystemHeteroCpuPolicy : Pos 7, 1 Bit
   +0x07f RunningNonRetpolineCode : Pos 0, 1 Bit
   +0x07f SpecCtrlSpare    : Pos 1, 7 Bits
   +0x07f SpecCtrl         : UChar
   +0x080 SystemCallNumber : Uint4B
   +0x084 ReadyTime        : Uint4B
   +0x088 FirstArgument    : Ptr64 Void
   +0x090 TrapFrame        : Ptr64 _KTRAP_FRAME
   +0x098 ApcState         : _KAPC_STATE
   +0x098 ApcStateFill     : [43] UChar
   +0x0c3 Priority         : Char
   +0x0c4 UserIdealProcessor : Uint4B
   +0x0c8 WaitStatus       : Int8B
   +0x0d0 WaitBlockList    : Ptr64 _KWAIT_BLOCK
   +0x0d8 WaitListEntry    : _LIST_ENTRY
   +0x0d8 SwapListEntry    : _SINGLE_LIST_ENTRY
   +0x0e8 Queue            : Ptr64 _DISPATCHER_HEADER
   +0x0f0 Teb              : Ptr64 Void
   +0x0f8 RelativeTimerBias : Uint8B
   +0x100 Timer            : _KTIMER
   +0x140 WaitBlock        : [4] _KWAIT_BLOCK
   +0x140 WaitBlockFill4   : [20] UChar
   +0x154 ContextSwitches  : Uint4B
   +0x140 WaitBlockFill5   : [68] UChar
   +0x184 State            : UChar
   +0x185 Spare13          : Char
   +0x186 WaitIrql         : UChar
   +0x187 WaitMode         : Char
   +0x140 WaitBlockFill6   : [116] UChar
   +0x1b4 WaitTime         : Uint4B
   +0x140 WaitBlockFill7   : [164] UChar
   +0x1e4 KernelApcDisable : Int2B
   +0x1e6 SpecialApcDisable : Int2B
   +0x1e4 CombinedApcDisable : Uint4B
   +0x140 WaitBlockFill8   : [40] UChar
   +0x168 ThreadCounters   : Ptr64 _KTHREAD_COUNTERS
   +0x140 WaitBlockFill9   : [88] UChar
   +0x198 XStateSave       : Ptr64 _XSTATE_SAVE
   +0x140 WaitBlockFill10  : [136] UChar
   +0x1c8 Win32Thread      : Ptr64 Void
   +0x140 WaitBlockFill11  : [176] UChar
   +0x1f0 Ucb              : Ptr64 _UMS_CONTROL_BLOCK
   +0x1f8 Uch              : Ptr64 _KUMS_CONTEXT_HEADER
   +0x200 ThreadFlags2     : Int4B
   +0x200 BamQosLevel      : Pos 0, 8 Bits
   +0x200 ThreadFlags2Reserved : Pos 8, 24 Bits
   +0x204 Spare21          : Uint4B
   +0x208 QueueListEntry   : _LIST_ENTRY
   +0x218 NextProcessor    : Uint4B
   +0x218 NextProcessorNumber : Pos 0, 31 Bits
   +0x218 SharedReadyQueue : Pos 31, 1 Bit
   +0x21c QueuePriority    : Int4B
   +0x220 Process          : Ptr64 _KPROCESS
   +0x228 UserAffinity     : _GROUP_AFFINITY
   +0x228 UserAffinityFill : [10] UChar
   +0x232 PreviousMode     : Char
   +0x233 BasePriority     : Char
   +0x234 PriorityDecrement : Char
   +0x234 ForegroundBoost  : Pos 0, 4 Bits
   +0x234 UnusualBoost     : Pos 4, 4 Bits
   +0x235 Preempted        : UChar
   +0x236 AdjustReason     : UChar
   +0x237 AdjustIncrement  : Char
   +0x238 AffinityVersion  : Uint8B
   +0x240 Affinity         : _GROUP_AFFINITY
   +0x240 AffinityFill     : [10] UChar
   +0x24a ApcStateIndex    : UChar
   +0x24b WaitBlockCount   : UChar
   +0x24c IdealProcessor   : Uint4B
   +0x250 NpxState         : Uint8B
   +0x258 SavedApcState    : _KAPC_STATE
   +0x258 SavedApcStateFill : [43] UChar
   +0x283 WaitReason       : UChar
   +0x284 SuspendCount     : Char
   +0x285 Saturation       : Char
   +0x286 SListFaultCount  : Uint2B
   +0x288 SchedulerApc     : _KAPC
   +0x288 SchedulerApcFill1 : [3] UChar
   +0x28b QuantumReset     : UChar
   +0x288 SchedulerApcFill2 : [4] UChar
   +0x28c KernelTime       : Uint4B
   +0x288 SchedulerApcFill3 : [64] UChar
   +0x2c8 WaitPrcb         : Ptr64 _KPRCB
   +0x288 SchedulerApcFill4 : [72] UChar
   +0x2d0 LegoData         : Ptr64 Void
   +0x288 SchedulerApcFill5 : [83] UChar
   +0x2db CallbackNestingLevel : UChar
   +0x2dc UserTime         : Uint4B
   +0x2e0 SuspendEvent     : _KEVENT
   +0x2f8 ThreadListEntry  : _LIST_ENTRY
   +0x308 MutantListHead   : _LIST_ENTRY
   +0x318 AbEntrySummary   : UChar
   +0x319 AbWaitEntryCount : UChar
   +0x31a AbAllocationRegionCount : UChar
   +0x31b SystemPriority   : Char
   +0x31c SecureThreadCookie : Uint4B
   +0x320 LockEntries      : Ptr64 _KLOCK_ENTRY
   +0x328 PropagateBoostsEntry : _SINGLE_LIST_ENTRY
   +0x330 IoSelfBoostsEntry : _SINGLE_LIST_ENTRY
   +0x338 PriorityFloorCounts : [16] UChar
   +0x348 PriorityFloorCountsReserved : [16] UChar
   +0x358 PriorityFloorSummary : Uint4B
   +0x35c AbCompletedIoBoostCount : Int4B
   +0x360 AbCompletedIoQoSBoostCount : Int4B
   +0x364 KeReferenceCount : Int2B
   +0x366 AbOrphanedEntrySummary : UChar
   +0x367 AbOwnedEntryCount : UChar
   +0x368 ForegroundLossTime : Uint4B
   +0x370 GlobalForegroundListEntry : _LIST_ENTRY
   +0x370 ForegroundDpcStackListEntry : _SINGLE_LIST_ENTRY
   +0x378 InGlobalForegroundList : Uint8B
   +0x380 ReadOperationCount : Int8B
   +0x388 WriteOperationCount : Int8B
   +0x390 OtherOperationCount : Int8B
   +0x398 ReadTransferCount : Int8B
   +0x3a0 WriteTransferCount : Int8B
   +0x3a8 OtherTransferCount : Int8B
   +0x3b0 QueuedScb        : Ptr64 _KSCB
   +0x3b8 ThreadTimerDelay : Uint4B
   +0x3bc ThreadFlags3     : Int4B
   +0x3bc ThreadFlags3Reserved : Pos 0, 8 Bits
   +0x3bc PpmPolicy        : Pos 8, 2 Bits
   +0x3bc ThreadFlags3Reserved2 : Pos 10, 22 Bits
   +0x3c0 TracingPrivate   : [1] Uint8B
   +0x3c8 SchedulerAssist  : Ptr64 Void
   +0x3d0 AbWaitObject     : Ptr64 Void
   +0x3d8 ReservedPreviousReadyTimeValue : Uint4B
   +0x3e0 KernelWaitTime   : Uint8B
   +0x3e8 UserWaitTime     : Uint8B
   +0x3f0 GlobalUpdateVpThreadPriorityListEntry : _LIST_ENTRY
   +0x3f0 UpdateVpThreadPriorityDpcStackListEntry : _SINGLE_LIST_ENTRY
   +0x3f8 InGlobalUpdateVpThreadPriorityList : Uint8B
   +0x400 SchedulerAssistPriorityFloor : Int4B
   +0x404 Spare28          : Uint4B
   +0x408 ResourceIndex    : UChar
   +0x409 Spare31          : [3] UChar
   +0x410 EndPadding       : [4] Uint8B
</pre>

So now the decomp is the following:
<pre>
nt!KeInitializeApc+0x10:
fffff801`50d41e9d 48f7d8          neg     rax                      rax = -NormalRoutine sets carry flag to one iff Normal not 0
 
fffff801`50d41ea4 481bc9          sbb     rcx,rcx                  rcx = 0 if NormalRoutine == 0 else -1
fffff801`50d41ea7 48234c2440      and     rcx,qword ptr [rsp+40h]  rcx = rcx & NormalContext
fffff801`50d41eac 48f7da          neg     rdx                      rdx = -Thread
fffff801`50d41eaf 1ac0            sbb     al,al                    al = 0 if Thread == 0 else -1
fffff801`50d41eb1 22442438        and     al,byte ptr [rsp+38h]    al = al & ApcMode
fffff801`50d41eb5 41884251        mov     byte ptr [r10+51h],al    (r10 = Apc)
fffff801`50d41eb9 49894a38        mov     qword ptr [r10+38h],rcx
fffff801`50d41ebd 41c6425200      mov     byte ptr [r10+52h],0
fffff801`50d41ec2 41c6420100      mov     byte ptr [r10+1],0
fffff801`50d41ec7 c3              ret
  NTKERNELAPI VOID KeInitializeApc (
    IN PRKAPC Apc,
    IN PKTHREAD Thread,
    IN KAPC_ENVIRONMENT Environment,
    IN PKKERNEL_ROUTINE KernelRoutine,
    IN PKRUNDOWN_ROUTINE RundownRoutine OPTIONAL,
    IN PKNORMAL_ROUTINE NormalRoutine OPTIONAL,
    IN KPROCESSOR_MODE ApcMode,
    IN PVOID NormalContext
    ){
   Apc->Type = 0x12;
   Apc->Size = 0x58;
   if (Environment == CurrentApcEnvironment) {
    Environment = Thread->ApcStateIndex;
   }
   // win64 calling convention requires 32 bytes of shadow space, so 0x28 is the fifth param 
   Apc->RundownRoutine = RundownRoutine;
   Apc->ApcStateIndex = Environment;
   Apc->Thread = Thread;
   Apc->NormalRoutine = NormalRoutine;
   Apc->KernelRoutine = KernelRoutine;
    }
</pre>

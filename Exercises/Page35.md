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

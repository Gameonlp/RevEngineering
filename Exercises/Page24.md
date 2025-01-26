As an exercise, you should decompile this function so that it looks more “natural” (as opposed to our literal translation):

```asm
01: sub_1000AE3B proc near
02: push edi
03: push esi
04: call ds:lstrlenA
05: mov edi, eax
06: xor ecx, ecx
07: xor edx, edx
08: test edi, edi
09: jle short loc_1000AE5B
10: loc_1000AE4D:
11: mov al, [edx+esi]
12: mov [ecx+esi], al
13: add edx, 3
14: inc ecx
15: cmp edx, edi
16: jl short loc_1000AE4D
17: loc_1000AE5B:
18: mov byte ptr [ecx+esi], 0
19: mov eax, esi
20: pop edi
21: retn
22: sub_1000AE3B endp
```
Original decomp:
```C
char *sub_1000AE3B (char *str)
{
 int len, i=0, j=0;
 len = lstrlenA(str);
 if (len <= 0) {
   str[j] = 0;
   return str;
 }
 while (j < len) {
   str[i] = str[j];
   j = j+3;
   i = i+1;
 }
 str[i] = 0;
 return str;
}
```
Logically duplicate code removal:
```C
char *sub_1000AE3B (char *str)
{
 int len, i=0, j=0;
 len = lstrlenA(str);
// if (len <= 0) {
//   str[j] = 0;
//   return str;
// }
 while (j < len) { // j = 0 -> !(len > j) == len <= 0; which makes the if + block equivalent to the while false branch + block (i = j = 0) 
   str[i] = str[j];
   j = j+3;
   i = i+1;
 }
 str[i] = 0;
 return str;
}
```
Variable reduction:
```C
char *sub_1000AE3B (char *str)
{
 //int len, i=0, j=0;
 int len, i=0;
 len = lstrlenA(str);
 //while (j < len) {
 //str[i] = str[j];
 //  j = j+3;
 while (i * 3 < len) { // j is equivalent to i * 3, as long as signed integer does not overflow
   str[i] = str[i * 3];
   i = i+1;
 }
 str[i] = 0;
 return str;
}
```
Loop rewriting (while -> for):
```C
char *sub_1000AE3B (char *str)
{
 int len, i=0;
 len = lstrlenA(str);
 //while (i * 3 < len) {
 //  str[i] = str[i * 3];
 //  i = i+1;
 //} The while loop is equivalent to a for loop (without the definition of i)
 for (; i * 3 < len; ++i) {
   str[i] = str[i * 3];
 }
 str[i] = 0;
 return str;
}
```
Final code:
```C
//char *sub_1000AE3B (char *str) making it readable
char *decode (char *str)
{
 int len, i=0;
 len = lstrlenA(str);
 for (; i * 3 < len; ++i) {
   str[i] = str[i * 3];
 }
 str[i] = 0;
 return str;
}
```

# strcpy using stdcall and int 0x80 legacy calling convention

### Intro:

I will be making a binary that

* Implements `strcpy` and `strlen` without a libc
* Allocates a single page of dynamic memory
* Copies a string into that memory

See [Quick and Dirty Assembly](2021-02-21-quick-and-dirty-assembly.md) and [Pure Binary in Assembly](2021-03-21-pure-assembly-binary.md)
### STDCALL Calling conventions

The calling convention for STDCALL is:

* Arguments are passed R to L onto the stack \(e.g., first argument is pushed last\)
* The callee cleans the stack
* preserve ebx, esi, edi, ebp, esp



### Legacy int 0x80 calling conventions

[https://en.wikibooks.org/wiki/X86\_Assembly/Interfacing\_with\_Linux](https://en.wikibooks.org/wiki/X86_Assembly/Interfacing_with_Linux)

|  | syscall |
| :--- | :--- |
| argument 1 | ebx |
| argument 2 | ecx |
| argument 3 | edx |
| argument 4 | esi |
| argument 5 | edi |
| argument 6 | ebp |
| syscall number | eax |

NOTE: BSD systems also allow the int 0x80 call with pushing values on the stack, and use the above as an alternate calling convention. [http://www.int80h.org/bsdasm/\#system-calls](http://www.int80h.org/bsdasm/#system-calls)

### Getting memory with mmap2 syscall

```c
void *mmap2(unsigned long addr, unsigned long length,
            unsigned long prot, unsigned long flags,
            unsigned long fd, unsigned long pgoffset)
```

Both the `flags` and `prot` parameters take constants in C to make our lives easier. We'll have to look through the Linux source code to find them.

```assembly
; https://elixir.bootlin.com/linux/latest/source/include/uapi/asm-generic/mman-common.h#L23 
%define PROT_READ 0x1
%define PROT_WRITE 0x2
%define MAP_ANONYMOUS 0x20
; https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/mman.h#L17
%define MAP_PRIVATE 0x2
```

Now we plug our parameters into the appropriate registers and call `int 0x80` 

```assembly
  mov ebx, 0
  mov ecx, 4096            ; length < page length (4k) results in a page being allocated anyway
  mov edx, PROT_READ
  or edx, PROT_WRITE       ; R/W permissions
  mov esi, MAP_ANONYMOUS
  or esi, MAP_PRIVATE      ; private and not file backed (just allocate memory, don't make it point to a file)
  mov edi, -1              ; no fd
  mov ebp, 0x0             ; no offset
  mov eax, 192             ; syscall 192 is mmap2, 90 is mmap but fails b/c it wants an argument struct
                           ; https://stackoverflow.com/questions/59923709/problem-trying-to-call-mmap-in-32-bit-i386-assembly#comment105970761_59923709
  int 0x80
```

### Calling strcpy

The man page for `strcpy` gives use the following function definition

```c
char *strcpy(char *restrict dest, const char *src);
```

The destination will be the freshly mapped memory. After the call to `mmap` the address of that page of memory is in `eax` . 

To make life easier, I defined a string in the data section for the `src` parameter.

```assembly
section .data
str1: db 'this is only a test', 0
```

Once again, we plug the parameters into the appropriate registers but this time we use `call`. Since only 2 parameters are needed, the others are ignored.

```assembly
  push str1
  push eax      ; eax is address from mmap
  call strcpy
```

### Detour into 32 bit function basics

#### Prologue

The beginning of a function contains a prologue that does the following

1. Save off the previous base pointer \(required\)
2. Set the stack pointer equal to the current base pointer \(required\)
3. Subtract N bytes from the stack pointer for any local variables \(if needed\)
4. Save any preserved registers that this function clobbers \(if needed\)

```assembly
; prologue example
  push ebp
  mov ebp, esp
  sub esp, 0xC 
  push ebx
```

#### The stack after the prologue

Using the `ebp` as a reference we can access the arguments, saved EIP, and any local variables. Remember the stack grows downward \(subtract = using stack space\)

```assembly
; more args here if needed
ebp + 0xC <- argument 2
ebp + 0x8 <- argument 1       
ebp + 0x4 <- Saved EIP
ebp + 0x0 <- Current function's base pointer
ebp - 0x4 <- local var 1
ebp - 0x8 <- local var 2 
; more local vars here if needed
ebp - N*(0x4) <- local var N & <-- ESP
```

#### Epilogue

The end of a function contains a epilogue that reverses the prologue

1. Reset any preserved registers \(if needed\)
2. Add N bytes back to stack to "clean" local vars \(if needed\)
3. Set stack pointer back to base pointer \(required\)
4. Reset previous base pointer \(required\)

```assembly
; epilogue example from above example
  pop ebx
  add esp, 0xC
  mov esp, ebp
  pop ebp  
```

### strcpy part 1 - Determining how many bytes to copy

In order to copy the string, we need to know exactly how many bytes the source string is. Calling `strlen` will give us that length. We have to also clean the stack afterward.

Note: 

> STDCALL functions are name-decorated with a leading underscore, followed by an @, and then the number \(in bytes\) of arguments passed on the stack. This number will always be a multiple of 4, on a 32-bit aligned machine. [https://en.wikibooks.org/wiki/X86\_Disassembly/Calling\_Conventions\#STDCALL](https://en.wikibooks.org/wiki/X86_Disassembly/Calling_Conventions#STDCALL)

```assembly
_strcpy@8:
  push ebp
  mov ebp, esp        ; End of prologue, no local vars or clobbered regs

  xor eax,eax

  mov edx, [ebp + 0xC]  ; param 2 src
  push edx
  call strlen
  add esp, 0x4
```

### strlen

The man page for `strlen` gives us the following function definition

```c
size_t strlen(const char *s);
```

My implementation for `strlen` does the following:

1. Clear the `eax` register, to represent the null character `\0`
2. Copy the maximum number of bytes into `edx` and `ecx` \(4k, and this is cheating\)
3. Search until we find a byte in the string that matches `eax` 
4.  Subtract `ecx` from `edx` to get the number of bytes
5. Return 4 since the CALLEE cleans the stack

```assembly
_strlen@4:
  push ebp
  mov ebp, esp
  push ecx
  push edi

  cld
  xor eax,eax
  mov ecx, 0x1000
  mov edx, ecx 
  mov edi, [ebp + 0x8]
  b:
  repne scasb
  je strlen_done

  strlen_done:
  std
  sub edx, ecx
  mov eax, edx
  
  pop edi
  pop ecx
  mov esp, ebp
  pop ebp
  ret 4
```

The funky line `repne scasb` is shorthand for:

* Compare the contents of the `al` register with the byte at pointed at in `rsi`, and then increment or decrement the pointer at `esi` \(scasb\)
* Repeat while those bytes are not equal \(repne\) and `ecx` is not 0

### strcpy part 2 - The copy loop

Now that we have the length, we can make a loop that copies the required bytes over.

```assembly
  mov ecx, eax
  mov edx, ecx
  mov edi, [ebp + 0x8]  ; param 1 dst
  mov esi, [ebp + 0xC]  ; param 2 src
  cld
  rep movsb
  std 
  mov esp, ebp
  pop ebp
  ret 8
```

The line `rep movsb` is shorthand for:

* move the byte pointed at by `esi` into the byte pointed at by `edi`\(movsb\)
* Repeat while `ecx` is not 0 \(rep\)

Since I have two 4 byte parameters, I need to issue a `ret 8`

The final code can be seen [https://github.com/mulpdev/practice/blob/master/asm-strcpy-calling-conventions/strcpy-cdecl.asm](https://github.com/mulpdev/practice/blob/master/asm-strcpy-calling-conventions/strcpy-cdecl.asm)




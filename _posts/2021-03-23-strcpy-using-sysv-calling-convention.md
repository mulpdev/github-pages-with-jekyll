# strcpy using sysv calling convention

### Intro:

I will be making a binary that

* Implements `strcpy` and `strlen` without a libc
* Allocates a single page of dynamic memory
* Copies a string into that memory

See \[Part 1\]\(\) and \[Quick and Dirty Assembly\]\(\)

### SYSV Calling conventions

The x86-64 ABI for linux can be found [https://refspecs.linuxfoundation.org/elf/x86\_64-abi-0.99.pdf](https://refspecs.linuxfoundation.org/elf/x86_64-abi-0.99.pdf)

From Appendix A.2.1 Linux Conventions



|  | syscall | function call |
| :--- | :--- | :--- |
| argument 1 | rdi | rdi |
| argument 2 | rsi | rsi |
| argument 3 | rdx | rdx |
| argument 4 | r10 | rcx |
| argument 5 | r8 | r8 |
| argument 6 | r9 | r9 |
| syscall number | rax | - |

Figure 3.14 shows the preserved registers (for simplicity we only care about general purpose. There are more)

* rbx, rsp, rbp, r12 - r15

### Getting memory with mmap syscall

```c
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
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

Now we plug our parameters into the appropriate registers and call `syscall` 

```assembly
  mov rdi, 0
  mov rsi, 4096           ; length < page length (4k) results in a page being allocated anyway
  mov rdx, PROT_READ
  or rdx, PROT_WRITE      ; R/W permissions
  mov r10, MAP_ANONYMOUS
  or r10, MAP_PRIVATE     ; private and not file backed (just allocate memory, don't make it point to a file)
  mov r8, -1              ; no fd
  mov r9, 0x0             ; no offset
  mov rax, 9              ; syscall 9 is mmap
  syscall
```

### Calling strcpy

The man page for `strcpy` gives use the following function definition

```c
char *strcpy(char *restrict dest, const char *src);
```

The destination will be the freshly mapped memory. After the call to `mmap` the address of that page of memory is in `rax` . 

To make life easier, I defined a string in the data section for the `src` parameter.

```assembly
section .data
str1: db 'this is only a test', 0
```

Once again, we plug the parameters into the appropriate registers but this time we use `call`. Since only 2 parameters are needed, the others are ignored.

```assembly
  mov rdi, rax      ; rax is address from mmap
  mov rsi, str1
  call strcpy
```

### strcpy part 1 - Determining how many bytes to copy

In order to copy the string, we need to know exactly how many bytes the source string is. Calling `strlen` will give us that length. 

```assembly
strcpy:
  xor rax, rax

  mov r12, rdi
  mov rdi, rsi
  call strlen
```

### strlen

The man page for `strlen` gives us the following function definition

```c
size_t strlen(const char *s);
```

My implementation for `strlen` does the following:

1. Clear the `rax` register, to represent the null character `\0`
2. Copy the maximum number of bytes into `r8` and `rcx` \(4k, and this is cheating\)
3. Search until we find a byte in the string that matches `rax` 
4.  Subtract `rcx` from `r8` to get the number of bytes

```assembly
strlen:
  xor rax, rax
  cld            ; increment rsi/rdi during REP
  mov r8, 0x1000 ; same size as mmap
  mov rcx, r8    
  repne scasb
  je strlen_done
  
  strlen_done:
  std          ; decrement rsi/rdi during REP
  sub r8, rcx
  mov rax, r8
  ret
```

The funky line `repne scasb` is shorthand for:

* Compare the contents of the `al` register with the byte at pointed at in `rsi`, and then increment or decrement the pointer at `rsi` \(scasb\)
* Repeat while those bytes are not equal \(repne\) and `rcx` is not 0

### strcpy part 2 - The copy loop

Now that we have the length, we can make a loop that copies the required bytes over.

```assembly
  mov rcx, rax
  mov rdi, r12
  cld            ; increment rsi/rdi during REP
  rep movsb
  std            ; decrement rsi/rdi during REP
  ret
```

The line `rep movsb` is shorthand for:

* move the byte pointed at by `rsi` into the byte pointed at by `rdi`\(movsb\)
* Repeat while `rcx` is not 0 \(rep\)

The final code can be seen [https://github.com/mulpdev/practice/blob/master/asm-strcpy-calling-conventions/strcpy-sysv.asm](https://github.com/mulpdev/practice/blob/master/asm-strcpy-calling-conventions/strcpy-sysv.asm)




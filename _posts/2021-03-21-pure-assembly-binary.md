# pure assembly binary

## Writing strcpy in pure assembly

I was given a suggestion to write strcpy in three different calling conventions: `SYSV`, `cdecl`, and `stdcall`. I decided to take that suggestion and run with it. Could I do it in pure assembly with no C library? 

It would require me to dynamically allocate memory, write the `strcpy` and `strlen` code, and also write code that would call it.

### Problem 1:  What is the very first instruction of a pure assembly binary?

I've never written a binary in pure assembly. I had only dealt with some shellcode, small library code, or inline assembly. So the first thing was figure out how to start properly besides exporting `_start` .

> ebp,%ebp sets %ebp to zero. This is suggested by the ABI \(Application Binary Interface specification\), to mark the outermost frame. Next we pop off the top of the stack. On entry we have argc, argv and envp on the stack, so the pop makes argc go into %esi. We're just going to save it and push it back on the stack in a minute. Since we popped off argc, %esp is now pointing at argv. The mov puts argv into %ecx without moving the stack pointer. Then we and the stack pointer with a mask that clears off the bottom four bits. Depending on where the stack pointer was it will move itlower, by 0 to 15 bytes. In any case it will make it aligned on an even multiple of 16 bytes. This alignment is done so that all of thestack variables are likely to be nicely aligned for memory and cache efficiency, in particular, this is required for SSE \(StreamingSIMD Extensions\), instructions that can work on vectors of single precision floating point simultaneously.
>
> [http://dbp-consulting.com/tutorials/debugging/linuxProgramStartup.html\#toc\_link7](http://dbp-consulting.com/tutorials/debugging/linuxProgramStartup.html#toc_link7)

I'm not interfacing with a C library, so I don't need to do anything with `argc`, `argv`, or `envp`

```text
section   .text
global    _start
_start:
  xor ebp, ebp
  and esp, 0xfffffff0
```

### Problem 2: How to allocate dynamic memory

In C, I would make a call to a one of the `alloc()` family of function calls. I can't do that here, so I had to figure out what those did behind the scenes to allocate dynamic memory. 

Turns out, it does a LOT of stuff for you. TL;DR is that it manages small, standard sized chunks of memory for you using one of many algorithms \(e.g., Doug Lea's allocator `dlmalloc`\). It gets the dynamic memory via the `mmap` syscall

Looking at the man page for mmap, it's documenting the glibc wrapper not the actual syscall.

> C library/kernel differences 
>
> This page describes the interface provided by the glibc mmap\(\) wrapper function. Originally, this function invoked a system call of the same name. Since kernel 2.4, that system call has been superseded by mmap2\(2\), and nowadays the glibc mmap\(\) wrapper function invokes mmap2\(2\) with a suitably adjusted value for offset.

The man page for `mmap2` provides the function definition for the syscall

```c
void *mmap2(unsigned long addr, unsigned long length,
            unsigned long prot, unsigned long flags,
            unsigned long fd, unsigned long pgoffset)
```

But wait, it also says

> This system call does not exist on x86-64.

A quick look at the [x86-64 syscall table](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall\_64.tbl) confirms that `mmap` is defined, but `mmap2` is not. So for x86-64, we must use the `mmap` syscall declaration

```c
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
```

The minimum allocation for an `mmap` call is typically the system page size. For x86/x86-64 that is 4kb.

### What's next?

Since I don't want to write my own allocator and memory manager, so I'm going to be extremely wasteful in these exercises. I'll document useful info in the following posts




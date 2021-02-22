---
title: Quick and Dirty Assembly
published: true
---

# Instruction References

AMD64 Architecture Programmer’s Manual Volume 3: General Purpose and System Instructions: https://developer.amd.com/resources/developer-guides-manuals/

Intel: https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-instruction-set-reference-manual-325383.pdf


# NASM and Intel syntax

https://www.nasm.us/doc/nasmdoc0.html

https://cs.lmu.edu/~ray/notes/nasmtutorial/


## Compile and link without glibc 

Entry point is `_start` by default

```assembly
section   .text
global    _start
_start:
```

```bash
nasm -f elf64 foo.asm -o foo.o
ld -m elf_x86_64 -entry=_start -o foo foo.o 
```

## Compile and link with glibc

Entry point is `main` by default

```assembly
section   .text
global    main
main:
```

```bash
nasm -f elf64 foo.asm -o foo.o
gcc -m64 -o foo foo.o
```

# AS and AT&T Syntax

From https://stackoverflow.com/a/7190511

```
gcc can use an assembly file as input, and invoke the assembler as needed. There is a subtlety, though:

    If the file name ends with ".s" (lowercase 's'), then gcc calls the assembler.
    If the file name ends with ".S" (uppercase 'S'), then gcc applies the C preprocessor on the source file (i.e. it recognizes directives such as #if and replaces macros), and then calls the assembler on the result.
```

## Compile and link without glibc 

Entry point is `_start` by default

```assembly
.text
.globl _start
_start:
```

```bash
gcc -c foo.s
ld -s -o foo foo.o
```

OR

```bash
gcc -m64 -nostdlib -Wl,-e_start fo.S -o foo
```

## Compile and link with glibc

Entry point is `main` by default

```assembly
.text
.globl main
main:
```

```bash
gcc -o write write.s
```

# Pure binary with no ELF format

```bash
gcc -c foo.c     
$ objcopy -O binary -j .text foo.o foo.bin
```

```bash
objcopy --only-section=.text --output-target binary foo.o foo.bin
```

```
gcc -Wl,--oformat=binary -o foo.bin foo.c
```

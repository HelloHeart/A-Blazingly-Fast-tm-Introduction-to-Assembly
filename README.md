# A Blazingly Fast(tm) Introduction to Assembly

View on: https://stackedit.io/app#

## The Important Bits

### What is Machine Language / Machine Code?

- A programming language made entirely out of raw bits

- `opcodes` are like functions, and `operands` are like arguments to those functions

- An opcode + all of its operands is called an `instruction`
	- you can think of an instruction as a single line of code in machine language

### What is an Instruction Set Architecture (ISA)?

You've definitely heard of them
- Some popular ISA's are x86, x86_64, ARM, MIPS, and RISC-V
	- x86_64 is the 64 bit version of x86, whereas x86 is 32 bits
		- you most likely run x86_64 on your current cpu
			- if you're on linux, you can check using `lscpu | grep Architecture`

Also called the Computer Architecture, an ISA determines things like:
- what registers and opcodes are available
- how computer memory is accessed
- how instructions are parsed from machine code
	- ex. `0xc3` is the `ret` instruction in x86, whereas the `ret` instruction doesn't exist in arm

***We'll be covering x86_64 for the rest of this workshop***

### What is assembly?
- Simply put, a human-readable version of machine code
- Two popular choices of syntax:
	- Intel syntax, aka the good one and the one we'll be using here
		- represents the bytes `48 83 7d f8 3f` like this: `cmp QWORD PTR [rbp-0x8],0x3f`
	- AT&T syntax, aka the questionable one you use when you want to scare your friends
		- represents the same bytes (`48 83 7d f8 3f`) like this: `cmpq $0x3f,-0x8(%rbp)`
			- important thing to note is how the operands are switched around, and that the `QWORD` is indicated by the `q` at the end of the `cmpq` mnemonic

### How Much Nicer is Assembly, Really?

![test](https://www.nayuki.io/res/a-fundamental-introduction-to-x86-assembly-programming/machine-code.svg)
> Image from https://www.nayuki.io/page/a-fundamental-introduction-to-x86-assembly-programming

> Note that the above diagram uses the at&t asm syntax

With any compiled binary named `x`, try running `objdump -M intel -d x`  
Objdump displays information about a file, and we use the following flags:
- -M intel
> output assembly instructions using intel syntax 
- -d
> display assembly instructions in addition to machine code

You should get some output like the following:

![File:Objdump-text-intel-black-wwright.png - wiki.ucalgary.ca](https://wiki.ucalgary.ca/images/8/81/Objdump-text-intel-black-wwright.png)
### Computer Memory

- When we talk about computer memory, we're usually talking about something called virtual memory. 
- we have `2^48` possible addresses, each able to store a single byte, meaning we can store `2^48` = 256 TB in virtual memory.

The most important part I'd like to reiterate in this section: ***Each memory address stores exactly one byte.***

### The Stack

- it's like a stack data structure (last in first out), except you can access and modify memory anywhere inside of it
	- so what actually gets pushed onto it?
- normally you push new data onto the stack, and it gets put at the highest non-used address that can fit everything (hence the "last in" part)
	- ex. if the stack is empty and you want to store an int (4 bytes), it stores it at `0xfffffffffffffffc`, then the next int would be stored at `0xfffffffffffffff8`

If you want to think of a program stack at a higher and much more useful level, you can think of it as a stack of stack frames, where stack frames are what we push onto and pop off the stack
-	we only ever work with the topmost (closest to the lowest address) stack frame

#### Stack Frames

- a stack frame is all the local data used by a called function
	- to be more specific, a stack frame holds a called function's saved registers, local variables, and passed parameters
- when you call a function, the current function creates a stack frame for it, large enough to fit all of the local information the function will access through its entire lifetime
	- this is the case for the x64 calling convention, but not for others
		- more on calling conventions later

If this is a bit confusing, it'll make more sense when we get to calling conventions

## Reading Assembly Instructions

### Syntax

The skeleton of most instructions you'll end up seeing:

`mnemonic ((metadata) operand1, (metadata) operand2, ...)`

where:
- () indicates an optional argument
- mnemonic = the operation you want to perform (add, mov, etc.)
- metadata = information about the operand
	- eg. the size you want to use, or information on memory layout

If you keep this format in mind, individual instructions become mostly trivial to understand.

### Mnemonics

You're not going to remember what all mnemonics do, so it's useful to have an instruction reference handy
- x86_64 opcodes and their respective mnemonics: http://ref.x86asm.net/coder64.html
- x86 opcodes and their respective mnemonics: http://ref.x86asm.net/coder32.html 

### Operands
An operand is generally one of the following: 
- a register
- an immediate (usually constants like the number 17)
- a label (named locations in memory)
- an arbitrary location in memory

When an instruction has two operands, say `add ebx, eax`, the first operand (`ebx` in this case) is usually the destination, and the second operand is usually the source
- this means that the result is usually stored in the first operand, but it's always good to check if you're not sure
- ex. in the instruction `add ebx, eax`, `add` takes the contents of `eax` and adds that to the contents of `ebx`, then stores the result in `ebx`

### Registers
You can think of registers as a bunch of pre-declared variables with set sizes.
- some registers are used by the cpu
	- the most important cpu-controlled registers to know are:
		-  the RIP register, which holds the address of the next instruction to be executed by the cpu (IP stands for Instruction Pointer) 
		- the RFLAGS register, which we'll mostly be using for  jumps (the `goto`'s of assembly)
-  other registers are used only by the program (these are called `General Purpose registers`)
	- RAX, RBX, RCX, RDX, RBP, RSI, RDI, RSP, R8, R9, ..., R15 are all 8-byte general purpose registers

If we don't need the entire register, we can decide to use a portion of it by using a different name.
- ex. with integers, which are 4 bytes long, you'll often find assembly code using `eax` instead of `rax`

![Microprocessor I: General Purpose Registers | by Nafisa Naznin | Medium](https://miro.medium.com/max/809/0*H8eaKdSdiGQHM_V7)
> from https://medium.com/@nafisanaznin24cse041/microprocessor-i-general-purpose-registers-31cd43f3d6bd

#### Important General Purpose Registers

RBP = the base pointer, it points to (aka, it holds the address of) the bottom of a stack frame
RSP = the stack pointer, it points to the top of the stack (aka the lowest (address used by the stack)

### The Instruction Pointer

Every time an instruction is finished executing, the cpu updates the instruction pointer (stored in the RIP register, so really it updates the cpu updates the RIP register) to point to the next instruction

We have special instructions to manipulate the RIP register
- ex. variations of the `jmp` operation, `call`, `ret`, etc.
- normal operations like `mov` and `add` don't work on it
- this is the register we often want to manipulate when creating an exploit
	- ex. stack overflow to overwrite an instruction pointer on the stack
		- you'll understand how this exploit works inside and out when we get into godbolt later on

### RFLAGS

- The top 32 bits of the RFLAGS register are all reserved (not in use at the moment), so we're pretty much working with EFLAGS (the lower 32 bits of RFLAGS)
- When we perform an operation, such as an addition or comparison, we can raise flags to indicate what happened in that operation
	- ex. if we call `add` and a register overflows, the overflow bit of the RFLAGS register will be set to 1 (setting a bit to 1 = raising a flag)
	- more important ex: if we use cmp to compare two values, RFLAGS will be updated accordingly to let us know if the two values were equal, or if one was bigger than the other

#### A note on little-endian

We store multiple-byte values (such as integers) in memory in an interesting way.

Say we want to store the 4-byte value `0xdeadbeef` into memory, be it the heap, the stack, or anything else

We do that by first storing the byte `0xef`, then `0xbe`, then `0xad`, and finally `0xde`
That is, we store the least significant byte closest to the lowest address

-	when we say "store in memory at address x" or "read from memory at address x", it usually means that we're accessing memory starting at x,  and then going up towards the higher addresses (0xffffffffffffffff)
	-	ex. if we want to store `0xdeadbeef` at `0xfffffffffffffffc`, it stores `0xef` at `0xfffffffffffffffc`, then `0xbe` at `0xfffffffffffffffd`, then `0xad` at `0xfffffffffffffffe`, and finally `0xde` at `0xffffffffffffffff`

So if you do a memory dump (in the direction of lower addresses -> higher addresses), `0xdeadbeef` will appear as `0xef 0xbe 0xad 0xde`

### Data Sizes 
- 1 `byte` = 8 bits = a single two-digit hex value (ex. 0xff)
- 1 `word` = 2 bytes = 16 bits = a 4-digit hex value (ex. 0xffff)
	- also called a `single word`
- 1 `dword` = 2 words = 4 bytes = 32 bits = an 8-digit hex value (ex. 0xffffffff)
- 1 `qword` = 4 words = 8 bytes = 64 bits = a 16-digit hex value (ex. 0xffffffffffffffff)
	- the `d` in `dword` stands for `double`, the `q` in `qword` stands for `quad`

Note: the hardware word size of a 64-bit computer is 8 bytes, a term and value we can safely ignore for the purposes of this workshop.

### Labels

- Some locations in memory contain things like hardcoded user strings or other global data
- labels are friendly names for these locations in memory

## What an ELF Binary Looks Like

### ELF vs PE

- windows uses PE, linux uses ELF
	- windows supports running ELF files via WSL, linux supports running PE files via Wine

### Program Layout of an ELF Binary (Simplified)

![text, data, bss, and dec](https://mirzafahad.github.io/img/text/memory.png)
> from https://softwareengineering.stackexchange.com/questions/386511/does-each-process-have-its-own-section-of-data-text-stack-and-heap-in-the-me

- lowest memory address = 0x0
- highest memory address = 0xffffffffffffffff
- .bss = global vars that get 0'd out by default
- .data = global vars that start with some predefined value you give it
- heap = where malloc reserves memory for your global data
- stack = where stackframes (aka called function information) are stored

### The x64 Calling Convention

The function `myfunc`:

```c
long myfunc(long a, long b, long c, long d,
            long e, long f, long g, long h)
{
    long xx = a * b * c * d * e * f * g * h;
    long yy = a + b + c + d + e + f + g + h;
    long zz = utilfunc(xx, yy, xx % yy);
    return zz + 20;
}
```

Produces this stackframe when called (you can ignore the red zone):

![Stack Frame x64](https://eli.thegreenplace.net/images/2011/08/x64_frame_nonleaf.png)
(code and image from https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64)

In general, the stack can be thought of as a sequence of stack frames, one after the other.

![Basic buffer overflow on 64-bit architecture | by null byte | Medium](https://miro.medium.com/max/578/1*Io2pbNYn8PeJCpSdNK8O8w.jpeg)
(Image from https://medium.com/@buff3r/basic-buffer-overflow-on-64-bit-architecture-3fb74bab3558)

## Godbolt

https://godbolt.org/

### A Basic Function
```c
#include  <stdio.h>

int add(int num) {
	return num + num;
}
  
int main() {
	int x = add(2);
	printf("%d\n", x);
}
```

### If Statements and Loops
```c
int main() {
	int x[4];
	for (int i = 0; i < 4; i++) {
		x[i] = i;
	}
  
	int i = 0;
	while (x[0] < 4) {
		x[i]++;
		i++;
	}
  
	if (x[0] == 4) {
		return  0;
	}
	else  if (x[0] > 5) {
		return  1;
	}
	else {
		return  2;
	}
}
```

### Pointers

```c
#include  <stdio.h>
  
void bad_swap(int *x, int *y) {
	// where's the bug?
	int z = *x;
	x = y;
	*y = z;
}
  
int main() {
	int a = 20;
	int b = 40;
	bad_swap(&a, &b);
	printf("I wonder what we get: %d, %d", a, b);
}
```

### Arrays

```c
#include  <stdio.h>
  
void param_pass(int x[4]) {
	*x = 0;
	// prints:
	// x: {0, 2, 3, 3}
	printf("x: {%d, %d, %d, %d}\n", x[0], x[1], x[2], x[3]);
}
  
int main() {
	int x[4] = {1, 2, 3, 4};
	x[0] = x[1];
	x[1] = *x;
	int *y = &x[3];
	x[2] = y - x;
	x[3] = *(x+2);
	param_pass(x);
}
```

### Structs

Important note: the compiler can add padding to struct members, making the size larger than expected

![Padding the struct: How a compiler optimization can disclose stack memory –  NCC Group Research](https://i0.wp.com/research.nccgroup.com/wp-content/uploads/2021/07/image.png?resize=770%2C325&ssl=1)
> image from https://research.nccgroup.com/2019/10/30/padding-the-struct-how-a-compiler-optimization-can-disclose-stack-memory/

```c
#include  <stdio.h>
#include  <stdlib.h>
#include  <string.h>
  
struct person {
	char *name;
	int age;
	int performance_scores[2];
};
  
int remote_person_mangler(struct person *p1) {
	p1->performance_scores[1] = 20;
	p1->performance_scores[1] *= 2;
	return p1->age;
}
  
int person_mangler(struct person p2) {
	p2.performance_scores[1] = 20;
	return p2.age;
}
  
int main() {
	struct person *p1 = malloc(sizeof(struct person));
	struct person p2;
	p1->name = malloc(31);
	strcpy(p1->name, "bobbert bobbington");
	p2.name = "bob";
	p1->age = 1000;
	p2.age = 18;
	p1->performance_scores[0] = 30;
	p2.performance_scores[0] = 10;
	int result1 = remote_person_mangler(p1);
	int result2 = person_mangler(p2);
}
```

## Short Intro to Ghidra
We'll get here if we have time

This will be a live demo of how to reverse a basic crackme from the ground up
-	from: https://crackmes.one/crackme/62072dd633c5d46c8bcbfd9b
-	password: `crackmes.one`


## Resources
### x86 asm
https://www.cs.virginia.edu/~evans/cs216/guides/x86.html
https://en.wikibooks.org/wiki/X86_Assembly/X86_Architecture

### ARM asm (and overall fantastic resource)
https://azeria-labs.com/writing-arm-assembly-part-1/

### Calling Conventions
https://aaronbloomfield.github.io/pdr/book/x86-32bit-ccc-chapter.pdf
https://aaronbloomfield.github.io/pdr/book/x86-64bit-ccc-chapter.pdf



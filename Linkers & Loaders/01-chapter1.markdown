[Linking vs. loading]

Linkers and loaders perform several related but conceptually separate actions:

• Program loading: Copy a program from secondary storage (which since about 1968 invariably means a disk) into main memory so it’s ready to run. In some cases loading just involves copying the data from disk to memory, in others it involves allocating storage, setting protection bits, or arranging for virtual memory to map virtual addresses to disk pages.

• Relocation: Compilers and assemblers generally create each file of object code with the program addresses starting at zero, but few computers let you load your program at location zero. If a program is created from multiple subprograms, all the subprograms have to be loaded at non-overlapping addresses. Relocation is the process of assigning load addresses to the various parts of the program, adjusting the code and data in the program to reflect the assigned
addresses. In many systems, relocation happens more than once. It’s quite common for a linker to create a program from multiple subprograms, and create one linked output program that starts at zero, with the various subprograms relocated to locations within the big program. Then when the program is loaded, the system picks the actual load address and the linked program is relocated as a whole to the load address.

• Symbol resolution: When a program is built from multiple subprograms, the references from one subprogram to another are made using symbols; a main program might use a square root routine called sqrt, and the math library defines sqrt. A linker resolves the symbol by noting the location assigned to sqrt in the library, and patching the caller’s object code to so the call instruction refers to that location.

[Two-pass linking]

When a linker runs, it first has to scan the input files to find the sizes of the segments and to collect the definitions and references of all of the symbols. It creates a segment table listing all of the segments defined in the input files, and a symbol table with all of the symbols imported or exported.

Using the data from the first pass, the linker assigns numeric locations to symbols, determines the sizes and location of the segments in the output address space, and figures out where everything goes in the output file.

The second pass uses the information collected in the first pass to control the actual linking process. It reads and relocates the object code, substituting numeric addresses for symbol references, and adjusting memory addresses in code and data to reflect relocated segment addresses, and writes the relocated code to the output file. It then writes the output file, generally with header information, the relocated segments, and symbol table information. If the program uses dynamic linking, the symbol table contains the info the runtime linker will need to resolve dynamic symbols. In many cases, the linker itself will generate small amounts of code or data in the output file, such as "glue code" used to call routines in overlays or dynamically linked libraries, or an array of pointers to initialization routines that need to be called at program startup time.

[Linking: a true-life example]

```c
// Source file m.c
extern void a(char *);
int main(int ac, char **av)
{
	static char string[] = "Hello, world!\n";
	a(string);
}

// Source file a.c
#include <unistd.h>
#include <string.h>
void a(char *s)
{
	write(1, s, strlen(s));
}
/////////////////////////////////////////////////////////////

//Object code for m.o
Sections:
Idx Name  Size      VMA       LMA       File off	Algn
0	.text 00000010  00000000  00000000  00000020	2 * *3
1	.data 00000010  00000010  00000010  00000030	2 * *3
Disassembly of section.text :
00000000 <_main> :
0 : 55 pushl %ebp
1 : 89 e5 movl %esp, %ebp
3 : 68 10 00 00 00 pushl $0x10
4 : 32.data
8 : e8 f3 ff ff ff call 0
9 : DISP32 _a
d : c9 leave
e : c3 ret
	...

//Object code for a.o
Sections :
Idx	Name  Size      VMA       LMA       File off	Algn
0	.text 0000001c  00000000  00000000  00000020	2 * *2
	CONTENTS, ALLOC, LOAD, RELOC, CODE
1	.data 00000000  0000001c  0000001c  0000003c	2 * *2
	CONTENTS, ALLOC, LOAD, DATA

Disassembly of section.text:
00000000 <_a> :
0 : 55 pushl %ebp
1 : 89 e5 movl %esp, %ebp
3 : 53 pushl %ebx
4 : 8b 5d 08 movl 0x8(%ebp), %ebx
7 : 53 pushl %ebx
8 : e8 f3 ff ff ff call 0
9 : DISP32 _strlen
d : 50 pushl %eax
e : 53 pushl %ebx
f : 6a 01 pushl $0x1
11 : e8 ea ff ff ff call 0
12 : DISP32 _write
16 : 8d 65 fc leal - 4(%ebp), %esp
19 : 5b popl %ebx
1a : c9 leave
1b : c3 ret
/////////////////////////////////////////////////////////////

//Selected parts of executable
Sections :
Idx Name  Size      VMA       LMA       File off  Algn
0.text 00000fe0  00001020  00001020  00000020  2 * *3
1.data 00001000  00002000  00002000  00001000  2 * *3
2.bss  00000000  00003000  00003000  00000000  2 * *3
Disassembly of section.text :
00001020 <start - c> :
...
1092 : e8 0d 00 00 00 call 10a4 <_main>
...
000010a4 <_main> :
10a4 : 55 pushl %ebp
10a5 : 89 e5 movl %esp, %ebp
10a7 : 68 24 20 00 00 pushl $0x2024
10ac : e8 03 00 00 00 call 10b4 <_a>
10b1 : c9 leave
10b2 : c3 ret
...
000010b4 <_a> :
10b4 : 55 pushl %ebp
10b5 : 89 e5 movl %esp, %ebp
10b7 : 53 pushl %ebx
10b8 : 8b 5d 08 movl 0x8(%ebp), %ebx
10bb : 53 pushl %ebx
10bc : e8 37 00 00 00 call 10f8 <_strlen>
10c1 : 50 pushl %eax
10c2 : 53 pushl %ebx
10c3 : 6a 01 pushl $0x1
10c5 : e8 a2 00 00 00 call 116c <_write>
10ca : 8d 65 fc leal - 4(%ebp), %esp
10cd : 5b popl %ebx
10ce : c9 leave
10cf : c3 ret
...
000010f8 <_strlen> :
...
0000116c <_write> :
...

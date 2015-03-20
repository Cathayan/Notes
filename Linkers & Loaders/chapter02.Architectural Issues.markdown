[Application Binary Interfaces]

In many cases, the linker has to do a significant part of the work involved in complying with the ABI. For example, if the ABI requires that each program contains a table of all of the addresses of static data used by routines in the program, the linker often creates that table, by collecting address information from all of the modules linked into the program. The aspect of the ABI that most often affects the linker is the definition of a standard procedure call.


[Procedure calls]

Within a procedure, data addressing falls into four categories: 

• The caller can pass arguments to the procedure.

• Local variables are allocated withing procedure and freed before the procedure returns.

• Local static data is stored in a fixed location in memory and is private to the procedure.

• Global static data is stored in a fixed location in memory and can be referenced from many different procedures. 


[Paging and Virtual Memory]

On most modern computers, each program can potentially address a vast amount of memory, four gigabytes on a typical 32 bit machine. Few computers actually have that much memory, and even the ones that do need to share it among multiple programs. Paging hardware divides a program’s address space into fixed size pages, typically 2K or 4K bytes in size, and divides the physical memory of the computer into page frames of the same size. The hardware conatins page tables with an entry for each page in the address space, as shown below:

![alt text](https://github.com/Cathayan/Notes/blob/master/Linkers%20%26%20Loaders/chapter02.page_mapping.png "Page Mapping")


A page table entry can contain the real memory page frame for the page, or flag bits to mark the page ‘‘not present.’’ When an application program attempts to use a page that is not present, hardware generates a page fault which is handled by the operating system. The operating system can load a copy of the contents page from disk into a free page frame, then let the application continue. By moving pages back and forth between main memory and disk as needed, the operating system can provide virtual memory which appears to the application to be far larger than the real memory in use.

Virtual memory comes at a cost, though. Individual instructions execute in a fraction of a microsecond, but a page fault and consequent page in or page out (transfer from disk to main memory or vice versa) takes several milliseconds since it requires a disk transfer. The more page faults a program generates, the slower it runs, with the worst case being thrashing, all page faults with no useful work getting done. The fewer pages a program needs, the fewer page faults it will generate. If the linker can pack related routines into a single page or a small group of pages, paging performance improves.

If pages can be marked as read-only, performace also improves. Read-only pages don’t need to be paged out since they can be reloaded from wherever they came from originally. If identical pages logically appear in multiple address spaces, which often happens when multiple copies of the same program are running, a single physical page suffices for all of the address spaces.


[Mapped files]

Virtual memory systems move data back and forth between real memory and disk, paging data to disk when it doesn’t fit in real memory. Originally, paging all went to ‘‘anonymous’’ disk space separate from the named files in the file system. Soon after the invention of paging, though, designers noticed that it was possible to unify the paging system and the file system by using the paging system to read and write named disk files. When a program maps a file to a part of the program’s address space, the operating system marks all of the pages in that part of the address space not present, and uses the file as the paging disk for that part of the address space, as in Figure 8. The program can read the file merely by referencing that part of the address space, at which point the paging system loads the necessary pages from disk.

![alt text](https://github.com/Cathayan/Notes/blob/master/Linkers%20%26%20Loaders/chapter02.mapping_a_file.png "Mapping A File")

There are three different approaches to handling writes to mapped files. The simplest is to map a file read-only (RO), so that any attempts to store into the mapped region fail, usually causing the program to abort. The second is to map the file read-write (RW), so that changes to the memory copy of the file are paged back to the disk by the time the file is unmapped. The third is to map the file copy-on-write (COW, not the most felicitous acronym). This maps the page read-only until the program attempts to store into the page. At that time, the operating system makes a copy of the page which is then treated as a private page not mapped from a file. From the program¡¯s point of view, mapping a file COW is very similar to allocating a fresh area of anonymous memory and reading the file's contents into that area, since changes the program makes are visible to that program but not to any other program that might have mapped the same file.


[Shared libraries and programs]

In nearly every system that handles multiple programs simultaneously, each program has a separate set of page tables, giving each program a logically separate address space. This makes a system considerably more robust, since buggy or malicious programs can¡¯t damage or spy on each other, but it potentially could cause performance problems. If a single program or single program library is in use in more than one address space, the system can save a great deal of memory if all of the address spaces share a single physical copy of the program or library. This is relatively straightforward for the operating system to implement . just map the executable file into each program¡¯s address space. Unrelocated code and read only data are mapped RO, writable data are mapped COW. The operating system can use the same physical page frames for RO and unwritten COW data in all the processes that map the file. (If the code has to be relocated at load time, the relocation process changes the code pages and they have to be treated as COW, not RO.)

Considerable linker support is needed to make this sharing work. In the executable program, the linker needs to group all of the executable code into one part of the file that can be mapped RO, and the data into another part that can be mapped COW. Each section has to start on a page boundary, both logically in the address space and physically in the file. When several different programs use a shared library, the linker needs to mark the each program so that when each starts, the library is mapped into the program’s address space.


[Position-independent code]

Shared libraries use Position Independent Code (PIC), code which will work regardless of where in memory it is loaded. All the code in shared libraries is usually PIC, so the code can be mapped read-only. Data pages still usually contain pointers which need relocation, but since data pages are mapped COW anyway, there’s little sharing lost.

For the most part, PIC is pretty easy to create. All three of the architectures we discussed in this chapter use relative jumps, so that jump instructions within the routines need no relocation. References to local data on the stack use based addressing relative to a base register, which doesn't need any relocation, either. The only challenges are calls to routines not in the shared library, and references to global data. Direct data addressing and the SPARC high/low register loading trick won¡¯t work, because they both require run-time relocation. Fortunately, there are a variety of tricks one can use to let PIC code handle inter-library calls and global data. We discuss them when we cover shared libraries in detail in Chapter 9 and 10.


[Demo Code]

```c
/**

 * map.c

 */

#include <stdio.h>

#include <stdlib.h>

#include <sys/types.h>

#include <unistd.h>



#define NSIZE   (5*(1<<20))

#define SLEEPT  10



long  gx[NSIZE];



const long c_gx = 1;





void callee_func()

{

  char  cc[NSIZE];



  for (int i=0; i<NSIZE; i++) {

    cc[i]  = 'c';

  }



  printf ("address of callee stack cc = %012p\n", &cc[0]);

  printf ("memory map file: /proc/%d/maps\n", getpid());

  printf ("sleeping %d...", SLEEPT);

  fflush (NULL);

  sleep (SLEEPT);



}


int main (int argc, char *argv[])

{

  char c1st = 'a';

  int  *px = malloc (NSIZE*sizeof(int));

  char* str = "this is a str";



  printf ("address of r/w data gx[0] = %012p\n", &gx[0]);

  printf ("address of r-only data c_gx = %012p\n", &c_gx);

  printf ("address of r-only string = %012p\n", str);

  printf ("address of heap px[0] = %012p\n", &px[0]);

  printf ("memory map file: /proc/%d/maps\n", getpid());

  printf ("sleeping %d...", SLEEPT);

  fflush (NULL);

  sleep (SLEEPT);

  char  c[NSIZE];



  for (int i=0; i<NSIZE; i++) {

    gx[i] = (long)i;

    px[i] = i;

    c[i]  = 'c';

  }

  char c2nd = 'a';


  printf ("address of  stack argv[0] = %012p\n", argv[0]);

  printf ("address of  stack c1st = %012p\n", &c1st);

  printf ("address of  stack c[0] = %012p\n", &c[0]);

  printf ("address of  stack c2nd = %012p\n", &c2nd);



  printf ("memory map file: /proc/%d/maps\n", getpid());

  printf ("sleeping %d...", SLEEPT);

  fflush (NULL);

  sleep (SLEEPT);



  callee_func();



  free (px);



  printf ("\ndone\n");

  exit (EXIT_SUCCESS);

}

/////////
address of r/w data gx[0] = 0x0008049b60
address of r-only data c_gx = 0x0008048820
address of r-only string = 0x0008048877
address of heap px[0] = 0x00b6b41008
memory map file: /proc/13513/maps
sleeping 10...

00146000-00161000 r-xp 00000000 fd:00 4910607    /lib/ld-2.5.so
00161000-00162000 r-xp 0001a000 fd:00 4910607    /lib/ld-2.5.so
00162000-00163000 rwxp 0001b000 fd:00 4910607    /lib/ld-2.5.so
001c2000-001c3000 r-xp 001c2000 00:00 0          [vdso]
001c3000-00319000 r-xp 00000000 fd:00 4910609    /lib/libc-2.5.so
00319000-0031b000 r-xp 00156000 fd:00 4910609    /lib/libc-2.5.so
0031b000-0031c000 rwxp 00158000 fd:00 4910609    /lib/libc-2.5.so
0031c000-0031f000 rwxp 0031c000 00:00 0 
08048000-08049000 r-xp 00000000 fd:00 5664072    /home/user/map
08049000-0804a000 rw-p 00000000 fd:00 5664072    /home/user/map
0804a000-0944a000 rw-p 0804a000 00:00 0 
b6b41000-b7f44000 rw-p b6b41000 00:00 0 
b7f56000-b7f57000 rw-p b7f56000 00:00 0 
bf4a3000-bf9a6000 rw-p bfafb000 00:00 0          [stack]

/////////
address of  stack argv[0] = 0x00bf9a58ef
address of  stack c1st = 0x00bf9a4067
address of  stack c[0] = 0x00bf4a4067
address of  stack c2nd = 0x00bf4a4066
memory map file: /proc/13513/maps
sleeping 10...

00146000-00161000 r-xp 00000000 fd:00 4910607    /lib/ld-2.5.so
00161000-00162000 r-xp 0001a000 fd:00 4910607    /lib/ld-2.5.so
00162000-00163000 rwxp 0001b000 fd:00 4910607    /lib/ld-2.5.so
001c2000-001c3000 r-xp 001c2000 00:00 0          [vdso]
001c3000-00319000 r-xp 00000000 fd:00 4910609    /lib/libc-2.5.so
00319000-0031b000 r-xp 00156000 fd:00 4910609    /lib/libc-2.5.so
0031b000-0031c000 rwxp 00158000 fd:00 4910609    /lib/libc-2.5.so
0031c000-0031f000 rwxp 0031c000 00:00 0 
08048000-08049000 r-xp 00000000 fd:00 5664072    /home/user/map
08049000-0804a000 rw-p 00000000 fd:00 5664072    /home/user/map
0804a000-0944a000 rw-p 0804a000 00:00 0 
b6b41000-b7f44000 rw-p b6b41000 00:00 0 
b7f56000-b7f57000 rw-p b7f56000 00:00 0 
bf4a3000-bf9a6000 rw-p bfafb000 00:00 0          [stack]

/////////
address of callee stack cc = 0x00befa4044
memory map file: /proc/13513/maps
sleeping 10...

00146000-00161000 r-xp 00000000 fd:00 4910607    /lib/ld-2.5.so
00161000-00162000 r-xp 0001a000 fd:00 4910607    /lib/ld-2.5.so
00162000-00163000 rwxp 0001b000 fd:00 4910607    /lib/ld-2.5.so
001c2000-001c3000 r-xp 001c2000 00:00 0          [vdso]
001c3000-00319000 r-xp 00000000 fd:00 4910609    /lib/libc-2.5.so
00319000-0031b000 r-xp 00156000 fd:00 4910609    /lib/libc-2.5.so
0031b000-0031c000 rwxp 00158000 fd:00 4910609    /lib/libc-2.5.so
0031c000-0031f000 rwxp 0031c000 00:00 0 
08048000-08049000 r-xp 00000000 fd:00 5664072    /home/user/map
08049000-0804a000 rw-p 00000000 fd:00 5664072    /home/user/map
0804a000-0944a000 rw-p 0804a000 00:00 0 
b6b41000-b7f44000 rw-p b6b41000 00:00 0 
b7f56000-b7f57000 rw-p b7f56000 00:00 0 
befa3000-bf9a6000 rw-p bf5fb000 00:00 0          [stack]







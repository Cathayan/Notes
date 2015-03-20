[Application Binary Interfaces]

In many cases, the linker has to do a significant part of the work involved in complying with the ABI. For example, if the ABI requires that each program contains a table of all of the addresses of static data used by routines in the program, the linker often creates that table, by collecting address information from all of the modules linked into the program. The aspect of the ABI that most often affects the linker is the definition of a standard procedure call.


[Procedure calls]

Within a procedure, data addressing falls into four categories: 

• The caller can pass arguments to the procedure.

• Local variables are allocated withing procedure and freed before the procedure returns.

• Local static data is stored in a fixed location in memory and is private to the procedure.

• Global static data is stored in a fixed location in memory and can be referenced from many different procedures. 


[Paging and Virtual Memory]

On most modern computers, each program can potentially address a vast amount of memory, four gigabytes on a typical 32 bit machine. Few computers actually have that much memory, and even the ones that do need to share it among multiple programs. Paging hardware divides a program’s address space into fixed size pages, typically 2K or 4K bytes in size, and divides the physical memory of the computer into page frames of the same size. The hardware conatins page tables with an entry for each page in the address space, as shown in Figure 7.







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







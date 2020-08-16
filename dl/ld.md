# How runtime dynamic linker works in Linux
How the runtime dynamic linker works in Linux had been a technical debt for me for a really long time until I decided to figure it out... TL;DR.
### 1. Prepare minimal test programs
l1.c, which will be compiled into a shared lib (so), defines a function which will be called by the main program (l2, see below):
	
	#include <stdio.h>
	int l1 = 10;
	int test() {
		printf("I am in libl1.so\n");
		return l1;
	}

compile with gcc:

	gcc -o libl1.so -fPIC -shared l1.c
	
l2.c, the main program which resides in the same directory as l1.c, calls a function defined in libl1.so:

	#include <stdio.h>
	extern int test();
	int main(void) {
		printf("test: %d\n", test());
	}

compile with gcc:

	gcc -o l2 -L. -ll1 l2.c -Wl,-rpath,"\$ORIGIN"

now the so libl1.so and executable file l2 have been created.

### 2. Analyze the main program with GDB

	objdump -d -j .text l2

output(only the main function, which is what I care about for this purpose):

	000000000000075a <main>:
	75a:	55                   	push   %rbp
	75b:	48 89 e5             	mov    %rsp,%rbp
	75e:	b8 00 00 00 00       	mov    $0x0,%eax
	763:	e8 c8 fe ff ff       	callq  630 <test@plt>
	768:	89 c6                	mov    %eax,%esi
	76a:	48 8d 3d a3 00 00 00 	lea    0xa3(%rip),%rdi        # 814 <_IO_stdin_used+0x4>
	771:	b8 00 00 00 00       	mov    $0x0,%eax
	776:	e8 a5 fe ff ff       	callq  620 <printf@plt>
	77b:	b8 00 00 00 00       	mov    $0x0,%eax
	780:	5d                   	pop    %rbp
	781:	c3                   	retq   
	782:	66 2e 0f 1f 84 00 00 	nopw   %cs:0x0(%rax,%rax,1)
	789:	00 00 00 
	78c:	0f 1f 40 00          	nopl   0x0(%rax)
	
the call to the `test()` function in libl1.so is at address 0x763:

	 callq  630 <test@plt>

which calls a function defined in the .plt section, as shown by:

	objdump -d -j .plt l2

output:

	0000000000000610 <.plt>:
	 610:	ff 35 a2 09 20 00    	pushq  0x2009a2(%rip)        # 200fb8 <_GLOBAL_OFFSET_TABLE_+0x8>
	 616:	ff 25 a4 09 20 00    	jmpq   *0x2009a4(%rip)        # 200fc0 <_GLOBAL_OFFSET_TABLE_+0x10>
	 61c:	0f 1f 40 00          	nopl   0x0(%rax)
	
	0000000000000620 <printf@plt>:
	 620:	ff 25 a2 09 20 00    	jmpq   *0x2009a2(%rip)        # 200fc8 <printf@GLIBC_2.2.5>
	 626:	68 00 00 00 00       	pushq  $0x0
	 62b:	e9 e0 ff ff ff       	jmpq   610 <.plt>
	
	0000000000000630 <test@plt>:
	 630:	ff 25 9a 09 20 00    	jmpq   *0x20099a(%rip)        # 200fd0 <test>
	 636:	68 01 00 00 00       	pushq  $0x1
	 63b:	e9 d0 ff ff ff       	jmpq   610 <.plt>

so, `test@plt` performs an indirect jump to the adress stored at 0x200fd0, shown below:

	objdump -s -j .got l2

output:

	 200fb0 a00d2000 00000000 00000000 00000000  .. .............
	 200fc0 00000000 00000000 26060000 00000000  ........&.......
	 200fd0 36060000 00000000 00000000 00000000  6...............
	 200fe0 00000000 00000000 00000000 00000000  ................
	 200ff0 00000000 00000000 00000000 00000000  ................

it can be seen that address stored at 0x200fd0 is 0x636, which just points to the next instruction:

	 636:	68 01 00 00 00       	pushq  $0x1
	 63b:	e9 d0 ff ff ff       	jmpq   610 <.plt>

and then goes to .plt at 0x610, which then pushes the value at 0x200fb8, which is just the second entry in the got: <_GLOBAL_OFFSET_TABLE_+0x8> onto the stack,
and then jumps to the address stored at 0x200fc0, which is the third entry in the got, but these values are all 0 in the binary image,
what are the actual values at runtime?


try stepping through with GDB:

	(gdb) r
	Starting program: /home/sz/workspace/test/src/l2 
	
	Breakpoint 1, main () at l2.c:4
	4		printf("test: %d\n", test());
	(gdb) si
	0x0000555555554763	4		printf("test: %d\n", test());
	(gdb) si
	0x0000555555554630 in test@plt ()
	(gdb) disass
	Dump of assembler code for function test@plt:
	=> 0x0000555555554630 <+0>:	jmpq   *0x20099a(%rip)        # 0x555555754fd0
	   0x0000555555554636 <+6>:	pushq  $0x1
	   0x000055555555463b <+11>:	jmpq   0x555555554610
	End of assembler dump.
	(gdb) x/xg 0x555555754fd0
	0x555555754fd0:	0x00007ffff7bd363a
	(gdb) disass 0x00007ffff7bd363a
	Dump of assembler code for function test:
	   0x00007ffff7bd363a <+0>:	push   %rbp
	   0x00007ffff7bd363b <+1>:	mov    %rsp,%rbp
	   0x00007ffff7bd363e <+4>:	lea    0x1c(%rip),%rdi        # 0x7ffff7bd3661
	   0x00007ffff7bd3645 <+11>:	callq  0x7ffff7bd3540 <puts@plt>
	   0x00007ffff7bd364a <+16>:	mov    0x20098f(%rip),%rax        # 0x7ffff7dd3fe0
	   0x00007ffff7bd3651 <+23>:	mov    (%rax),%eax
	   0x00007ffff7bd3653 <+25>:	pop    %rbp
	   0x00007ffff7bd3654 <+26>:	retq   
	End of assembler dump.
	(gdb) 
	
hum... the correct address of the `test()` function was already stored in the got even at the first time this function is called, that's
not how the online materials describe about dynamic liner: it resolves a funcion in a shared lib the first time it gets called. why?

### 3. Analyze the so with GDB

continuing with the debug session above, step with single instruction:

	(gdb) si
	test () at l1.c:3
	3	int test() {
	(gdb) si
	0x00007ffff7bd363b	3	int test() {
	(gdb) 
	5		printf("I am in libl1.so\n");
	(gdb) si
	0x00007ffff7bd3645	5		printf("I am in libl1.so\n");
	(gdb) si
	0x00007ffff7bd3540 in puts@plt () from /home/sz/workspace/test/src/libl1.so
	(gdb) disass 0x00007ffff7bd3540
	Dump of assembler code for function puts@plt:
	=> 0x00007ffff7bd3540 <+0>:	jmpq   *0x200ad2(%rip)        # 0x7ffff7dd4018
	   0x00007ffff7bd3546 <+6>:	pushq  $0x0
	   0x00007ffff7bd354b <+11>:	jmpq   0x7ffff7bd3530
	End of assembler dump.

similar to that of l2 above, check the content of got at runtime:

	(gdb) x/xg 0x7ffff7dd4018
	0x7ffff7dd4018:	0x00007ffff7bd3546

so, it's:

	 0x00007ffff7bd3546 <+6>:	pushq  $0x0

which means it has not been resolved yet and it will call into the dynamic linker for that, according to the .plt section:

	0000000000000530 <.plt>:
	 530:	ff 35 d2 0a 20 00    	pushq  0x200ad2(%rip)        # 201008 <_GLOBAL_OFFSET_TABLE_+0x8>
	 536:	ff 25 d4 0a 20 00    	jmpq   *0x200ad4(%rip)        # 201010 <_GLOBAL_OFFSET_TABLE_+0x10>
	 53c:	0f 1f 40 00          	nopl   0x0(%rax)
	
	0000000000000540 <puts@plt>:
	 540:	ff 25 d2 0a 20 00    	jmpq   *0x200ad2(%rip)        # 201018 <puts@GLIBC_2.2.5>
	 546:	68 00 00 00 00       	pushq  $0x0
	 54b:	e9 e0 ff ff ff       	jmpq   530 <.plt>
		
and .got.plt:

	Contents of section .got.plt:
	 201000 180e2000 00000000 00000000 00000000  .. .............
	 201010 00000000 00000000 46050000 00000000  ........F.......

the function `puts()` should be stored at the forth entry of the got, it's 0x546 in the binary image, but at run time, it's relocated.
but with GDB, the runtime got can be shown:

	(gdb) x/4xg 0x7ffff7dd4000
	0x7ffff7dd4000:	0x0000000000200e18	0x00007ffff7ff5000
	0x7ffff7dd4010:	0x00007ffff7dec6d0	0x00007ffff7bd3546

so the address in the binary image 0x546 has been relocated to 0x00007ffff7bd3546, and it just calls .plt this time, which, according to the code in
the .plt section, pushes the value stored at <_GLOBAL_OFFSET_TABLE_+0x8>: 0x00007ffff7ff5000(which is actually the address of the `link_map` for libl1.so in the code of the dynamic linker) onto the stack
and jumps to the address stored at <_GLOBAL_OFFSET_TABLE_+0x10>: 0x00007ffff7dec6d0(which is the adress of the function of the dynamic linker which resolves the symbol `puts`). 

after the symbol is resolved, it will fixup the got of libl1.so, i.e. replace the address 0x00007ffff7bd3546 with the address just resolved
so that next time the function gets called it will jump directly to the actual address:

	(gdb) x/4xg 0x7ffff7dd4000
	0x7ffff7dd4000:	0x0000000000200e18	0x00007ffff7ff5000
	0x7ffff7dd4010:	0x00007ffff7dec6d0	0x00007ffff7862a30

, and jump to the address so that the funtion gets called, this way, the caller doesn't even notice it performed the resolution(except the first function call
is slower).

### 4. Conclusion
It turns out that for the main program, even if it's of DYN type, the functions it calls get resolved at startup all at once(not LAZY), but functions in shared libs which the main program depends on will be resolved when first called (LAZY). 
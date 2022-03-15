## Low-level hacks: Mailboxes, ATAGS, and backtraces.

#### Introduction

The last few labs have involved writing some pretty tricky code. Today's lab
is more of an "odds and ends" types of lab that covers simple, but very 
important, tricks in low level systems programming. In today's lab, we focus 
on building: 

1. **ATAGS**: ATAGS (which stands for ARM-Tags) are used to carry information from 
   boot code to the kernel. ATAGS are a way to pass useful information such as 
   memory size from one important place (boot code) to another important place 
   (the kernel). For instance, when our hardware (i.e. our pi) boots up, it 
   loads `kernel.img`. (Remember, `kernel.img` is like the "brain" of our pi). 
   The internal bootloader passes parameters to `kernel.img` using ATAGS.

2. **Mailboxes**: The Mailbox Peripheral is a peripheral that facilitates 
   communication between the CPU (Central Processing Unit) and the GPU 
   (Graphics Processing Unit). Mailboxes give the pi a way to send messages
   to the GPU and to recieve a response. You can think of a Mailbox as the 
   main entry point into the GPU

3. **Stack Backtraces**: A Stack Backtrace is a report of the actice stack 
   frames at a certain point during the execution of a program. Up until 
   now, whenever you have gotten an assertion error or print a message, you
   only have information about the file, function, and, line, that are 
   directly corresponding to wherever the assertion is at. Without a stack 
   backtrace it can be hard to figure out what is going on given you don't
   have inofrmation about the caller(s). Today, you will write a simple
   backtrace implementation that walks back up the stack and gets the current
   callers. 


#### 1. ATAGS

We have a simple explanation in [atags](code-atags/README.md).  You sohuld
fill in the missing code in `code-atags/atags.h`) so that you can see how
they work and use them to get the memory size of the pi.

Copy this header to `libpi/src`.

#### 2. mailboxes

We have a simple explanation in [mailbox](code-mbox/README.md).
You should fill in the missing code in `code-mbox/mbox.h`) so that you
can see how they work and use them to get the memory size of the pi.


Note: the serial number should be different across all pi's.  For the pi zero, I was
getting 0 for the model number.

Copy this header to `libpi/src`.

#### 3. Increase your pi memory to 496MB

Look in [increase memory](increase-mem/README.md) to see how to 
increase your pi memoyr and get it up to 496MB.

The program in the directory should pass.

#### 4. Implement stack backtraces

The CS107E website has a nice writeup on 
[building backtraces](http://cs107e.github.io/assignments/assign4/).
This [memory diagram](http://cs107e.github.io/labs/lab4/images/stack_abs.html),
in particular, will be of use to you!

Implement this and show it works!  (We'll push some code in a bit).

There is a trivial program in `code-backtrace` to (weakly) test it.
It manually prints the call stack --- when you print the backtrace it
should match this (different formatting is ok).

### Extension: use the exception debugging information to do backtraces

We use the simple, standard way to do backtraces.  However, if you
look at the code, it adds a reasonable amount of overhead in terms of
additional instructions (especially loads and stores).  It's possible
to instead use the exception tables that C++ will generate to handle 
exceptions.  To do this you need to (1) get them emitted and (2) figure
out their format.  There's a bunch of different blog posts on doing this
but I didn't track down the details, so would be very interested!

### Extension: exception backtraces + gprof

If we get a weird exception, we'd like to print out where it the original
code it came from (besides just the exception pc) --- the issue here is
that you need to get the original frame pointer so we can walk backwards.
This shouldn't require too much work, but is useful.

Once you have this working, it's good to go back and add it to `gprof`
so that the profiling it gives is a bit more useful.   You'd just go
back the call stack two or three deep.  Note: that `libpi` currently
does not use the flags we need so let me know when you get here.

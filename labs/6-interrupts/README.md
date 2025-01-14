## Interrupts.

#### Introduction

In this lab, we will be working on implementing **interrupts**, which are
at the heart of operating system design. The kernel is responsible for 
keeping track of processes (which may be running on multiple devices) and
ensuring that they run correctly, and to completion. The kernel's job
ends up becoming that of a master scheduler who is responsible for 
overseeing everything that is happening on the machine. Managing everything
that is going on is no easy task. 

One way to understand process management is to think about process state. 
Process state tracks the _state_ of any given process, and allows the 
kernel to make important scheduling decisions about which process should 
run next. Most importantly, the scheduler needs to decide which process 
should have priority to run. The scheduler by might begin by running one
process--called, say, Process B--but will give priority to another 
process--called, say, Process A--if the second process needs immediate 
attention. In other words, the scheduler can choose to _interrupt_ 
Process A by forcing that process to stop running for the time being. 
The scheduler would do this by moving Process A from the running set to 
the ready queue. Here is a diagram to model this:

<table><tr><td>
  <img src="images/process-lifecycle-diagram.png" width="700"/>
</td></tr></table>

The key idea with interrupts is that we can manage the execution of 
multiple, concurrent processes by ensuring that processes interrupt
one another whenever they need immediate attention. 

This idea of interrupts can be extended further when thinking about how 
an operating system can manage multiple devices at once. The operating
system needs to keep track of what happens to each of these devices. In 
the case something "important" happens to one of these devices, the OS
needs to know immediately so it can appropriately deal with the event 
that just happened and give priority to this device. 

An operating system could try to constantly check what it is happening 
to each of the various devices using a technique known as polling, but 
the system would end up wasting all its time on these checks which 
would make these checks fruitless. If the OS only checked the status 
of these devices periodically then it would only learn about important
changes in the devices well after they happened. Interrupts solve this 
problem by ensuring that the OS can conduct its normal operations, and
that it will be interrupted as soon as an interesting event happens on 
a device it cares about. 

Of course, there are many different types of "interesting" events that
could happen on a device or to a process. When an interesting event does 
happen, the OS wants to know exactly what it should do in response to 
the event that just happened. This is where the idea of an interrupt
handler comes into play. 

An **interrupt handler** is a piece of code associated with a specific
interrupt condition. The interrupt handler is responsible for responding
to the event that just happened by performing the operations that are
necessary in order to service the interrupt. For instance, a timer 
interrupt (which happens when a given amount of time elapses) will 
require a different response than an exception interrupt (which happens
when something goes wrong in a program, such as when we divide by zero). 
Using an interrupt, the OS can ensure that when something important 
happens, it can interrupt the normal code that is running, run the 
interrupt handler for the event that just happened, and then jump back
to the normal code that was running before. 

Interrupts also permit you to write code that may not complete promptly, by giving
you the ability to run it for a given amount, and then interrupt
(pre-empt) it and switch to another thread of control. Operating systems
and runtime systems use this ability to make a pre-emptive threads
package and, later, user-level processes.  The timer interrupt we do
for today's lab will give you the basic framework to do this.

Traditionally interrupts are used to refer to transfers of control
initiated by "external" events (such as a device or timer-expiration).
These are sometimes called asynchronous events.  Exceptions are
more-or-less the same, except they are synchronous events triggered by the
currently running thread (often by the previous or current instruction,
such as a access fault, divide by zero, illegal instruction, etc.)
The framework we use will handle these as well; and, in fact, on the
ARM at a mechanical level there is little difference.

Like everything good, interrupts come at a cost: this will be our
first exposure to race conditions.  If both your regular code and the
interrupt handler read / write the same variables, you have a problem. 
You can have race conditions where both the regular code and the interrupt
handler can overwrite each other's values. In order to avoid this problem, 
we need to save any important values from the regular code that was 
executing, before we switch to the code in the interrupt handler. This
"saving-before-switching" idea is the idea behind what is known as a 
**context switch**. In a context switch, we switch from one piece of code
and all of its relevant variables to another piece of code and all of 
its relevant variables. For this lab, we will be context switching 
between the context of the regular C code and the context of the 
interrupt handler. Here is a diagram illustrating this:

<table><tr><td>
  <img src="images/interrupts-diagram.png" width="600"/>
</td></tr></table>

-----------------------------------------------------------------------------

#### Implementation Notes

***Make sure you read***: 
  - The [PRELAB](PRELAB.md), which has the readings and sketches
    the `kmalloc` implementation.
  - [INTERRUPTS](INTERRUPTS.md): this is a cheat sheet of useful page
    numbers and some notes on how the ARMv6 does exceptions.
  - [caller-callee registers](../../guides/caller-callee/README.md):
    this shows a cute trick on how to derive which registers `gcc` treats
    as caller (must be saved by the caller if it wants to use them after
    a procedure call) and callee (must be saved by a procedure before
    it can use them and restored when it returns).

Big picture for today's lab:

   0. We'll show how to setup interrupts / exceptions on the r/pi.
   Interrupts and exceptions are useful for responding quickly to devices,
   doing pre-emptive threads, handling protection faults, and other things.

   1. We strip interrupts down to a fairly small amount of code.  You will
   go over each line and get a feel for what is going on.  

   2. You will use timer interrupts to implement a simple but useful
   statistical profiler similar to `gprof`.  As is often the case,
   because the r/pi system is so simple, so too is the code we need to
   write, especially as compared to on a modern OS.

The good thing about the lab is that interrupts are often made very
complicated or just discussed so abstractly it's hard to understand them.
Hopefully by the end of today you'll have a reasonable grasp of at
least one concrete implementation.  If you start kicking its tires and
replacing different pieces with equivalant methods you should get a
pretty firm grasp.

### Deliverables:

Turn-in:

  1. `timer-int`: give the shortest time between timer interrupts you can
     make, and two ways to make the code crash.

  2. Build a simple system call: show your `1-syscall` works.

  3. Implement `gprof` (in the `2-gprof` subdirectory).   You should
     run it and show that the results are reasonable.  Explain why it
     spends all the time in the uart code.

----------------------------------------------------------------------------
### Part 0: timer interrupts.

Look through the code in `timer-int`, compile it, run it.  Make sure
you can answer the questions in the comments.  We'll walk through it
in class.

----------------------------------------------------------------------------
### Part 1: make a simple system call.

One we can get timer exceptions, we (perhaps surprisingly) have enough
infrastructure to make trivial system calls.   Since we are already
running in supervisor mode, these are not that useful as-is, but making
them now will show how trivial they actually are.  In particular, look in
`1-syscall` and write the needed code in:

  1. `interrupts-asm.S`: you should be able to largely rip off the timer interrupt
     code to forward system call.  NOTE: a huge difference is that we are
     already running at supervisor level, so all registers are live.  You need
     to handle this differently.

  2. `syscall.c`: finish the system call vector code (should just be a few lines).
     You want to act on system call 1 and reject all other calls with a `-1`.

This doesn't take much code, but you will have to think carefully about which
registers need to be saved, etc.

-----------------------------------------------------------------------------
### Part 2: Using interrupts to build a profiler.

The nice thing about doing everything from scratch is that simple things are simple
to do.  We don't have to fight a big OS that can't get out of its own way.   

Today's lab is a good example: implementing a statistical profiler.  The basic intuition:
   1. Setup timer interrupts so that we get them fairly often and at consistent intervals.
   2. At each interrupt, record which location in the code we interrupted.  (I.e., where
      the program counter is.)
   3. Over time, we will interrupt where your code spends most of its time more often.

The implementation will take about 30-40 lines of code in total.  You'll build two things:
 1. A `kmalloc` that will allocate memory.  We will not have a `free`, so `kmalloc` is
   trivial: have a pointer to where "free memory" starts, and increment a counter based
   on the requested size.
 2. Use `kmalloc` to allocate an array at least as big as the code.
 3. In the interrupt handler, use the program counter value to index into this array
    and increment the associated count.  NOTE: its very easy to mess up sizes.  Each
    instruction is 4 bytes, so you'll divide the `pc` by 4.  
 4. Periodically you can print out the non-zero values in this array along with the 
   `pc` value they correspond to.  You should be able to look in the disassembled code
   (`gprof.list`) to see which instruction these correspond to.

Congratulations!  You've built something that not many people in the Gates building
know how to do.

-----------------------------------------------------------------------------
### lab extensions:

There's lots of other things to do:

  1. Mess with the timer period to get it as short as possible.

  2. To access the banked registers can use load/store multiple (ldm, stm)
    with the caret after the register list: `stmia sp, {sp, lr}^`.

  3. Using a suffix for the `CPSR` and `SPSR` (e.g., `SPSR_cxsf`) to 
	specify the fields you are [modifying]
     (https://www.raspberrypi.org/forums/viewtopic.php?t=139856).

  4. New instructions `cps`, `srs`, `rfe` to manipulate privileged state
   (e.g., Section A4-113 in `docs/armv6.pdf`).   So you can do things like

            cpsie if @ Enable irq and fiq interrupts

            cpsid if @ ...and disable them

-----------------------------------------------------------------------------
### Supplemental documents

There's plenty to read, all put in the `docs` directory in this lab:
 
  1. If you get confused, the overview at `valvers` was useful: (http://www.valvers.com/open-software/raspberry-pi/step04-bare-metal-programming-in-c-pt4)

  2. We annotated the Broadcom discussion of general interrupts and
  timer interrupts on the pi in `7-interrupts/docs/BCM2835-ARM-timer-int.annot.pdf`.
  It's actually not too bad.

  3. We annotated the ARM discussion of registers and interrupts in
  `7-interrupts/docs/armv6-interrupts.annot.pdf`.

  4. There is also the RealView developer tool document, which has 
  some useful worked out examples (including how to switch contexts
  and grab the banked registers): `7-interrupts/docs/DUI0203.pdf`.

  5. There are two useful lectures on the ARM instruction set.
  Kind of dry, but easier to get through than the arm documents:
  `7-interrupts/docs/Arm_EE382N_4.pdf` and `7-interrupts/docs/Lecture8.pdf`.

If you find other documents that are more helpful, let us know!

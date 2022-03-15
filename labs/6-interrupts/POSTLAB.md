## Interrupts: Redux

#### Student Perspective

This lab felt like one of the first real-world operating system labs, which 
was super exciting! Interrupts are a key part of any functional operating 
system so being able to implement the infratructure to make them work was
very exciting. One of the key things I learned in this lab was the importance
of using _assembly_ as opposed to _C-code_ in certain contexts. 

------------------------------------------------------------------------------

#### Key Points from the Lab

##### 1. The Benefits of Assembly

For most of us, our last tangible experience with assembly was in CS107 when 
doing Binary Bomb/SecureVault. The resulting relationship with assembly is either one of
love or one of hate. In this lab, we learn the importance of using assembly, 
and why sometimes we have no other option _but_ to use assembly. There is some
functionality that it is simply not possible to express in C. In the case of
interrupts, it's important to write them in assembly in order to be sure that
we preserve all registers correctly. 

In the case of interrupts, this is especially the case because we need to 
work with special registers that are otherwise not available in C, and we 
need to guarantee that instructions execute in the order that we want them
to. In the case of interrupts, we need to be able to access stack pointers 
and registers.

With standard C-code, we can't guarantee that the compiled version of the code
is exactly what we expect it to be. This is because, at times, the compiler will
try to optimize code and be more efficient. In most of the programs we have 
written thus far, this isn't an issue. However, as we progress through the
code, we will need to rely on assembly more in order to be able to write 
functional low-level code. 

##### 2. Handling Different Types of Interrupts

In our implementation, we handle interrupts using an interrupt table with 
different branches for the different types of interrupts we might encounter. 
Here is the interrupt table inside of `interrupts-asm.S`:

    .globl _interrupt_table
    .globl _interrupt_table_end
    _interrupt_table:
      ldr pc, _reset_asm
      ldr pc, _undefined_instruction_asm
      ldr pc, _software_interrupt_asm
      ldr pc, _prefetch_abort_asm
      ldr pc, _data_abort_asm
      ldr pc, _reset_asm
      ldr pc, _interrupt_asm

One of the key benefits of this type of implementation is that we can 
leverage assembly in order to ensure that interrupts are implemented
correctly. However, one of the key disadvantages is that we would need to
modify this interrupt table and the surrounding `.S` file manually every 
time we want to add a new type of interrupt.  

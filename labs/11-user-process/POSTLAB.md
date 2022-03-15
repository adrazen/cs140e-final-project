## User Process & Equivalence: Redux

#### Student Perspective
This lab combined many, many concepts from prior labs. At times, it can
be easy to focus in on just the function you're currently writing, but
take time to consider how the work you've done over the last few weeks
built up to this lab! I'd 100% suggest asking Dawson and course staff 
questions, regardless of how big or small the topic is that you're 
puzzling over.

------------------------------------------------------------------------------

#### Key Points from the Lab

##### 1. Why single-stepping?
This lab built off of the debug-hardware lab in order to implement single-stepping
with breakpoints. As you probably know from your debug work outside of this class, 
single stepping is simply a debug mechanism that allows you to move instruction by 
instruction through your code (a single step at a time). In this lab, we combine 
single-stepping with our class staple of equivalence checking (comparing your code to 
the rest of the class and course staff in order to verify its correctness). Today for example,
you generated hashes of the register values at every step in your code, using the hash
from the prior step, in order to compare the final hash after the final instruction.
This type of single step hash equivalence checking is actually a cool security 
feature because in the event of some sort of breach, there is no way to avoid
modifying the hashes produced by single stepping. It will be obvious that 
something in the system changed, due to the hashes changing after the breach.

##### 2. Switching to user level
We used the `cps` instruction in order to switch our processor mode to user level,
before  continuing on to the function we want to run. However, there were a few
steps in between the `cps` and updateing `pc` to hold the function. In particular,
we needed to use a prefetch flush, directly after `cps`. Prefetch flush ensures that
the preceding instructions occur before proceeding - we want to make sure that
our switch to user level sticks! We then clear a lot of registers. When switching
privilege modes, it makes sense from a security standpoint to avoid allowing a 
lower privilege process access to data from a higher privileged process.

##### 3. Trampolines
These assembly functions are simply facilitators for other routines. We are able
to save relevant registers (e.g with `push`) before branching to a routine. In 
particular, since most of our interrupt handlers require similar faciliation,
we can use the same trampoline for most of them. When a routine complete, we
can then `pop` the saved register values and continue on.

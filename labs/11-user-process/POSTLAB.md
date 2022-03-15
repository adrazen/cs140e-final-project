## User Process & Equivalence: Redux

#### Student Perspective

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
Security:  When transition from kernel to user level, u wanna clear registers so data isn't accessible from different privilege level

prefetch flush after cps to ensure the switch "sticks"

what is cpsr what are we doing here

##### 3. Trampolines

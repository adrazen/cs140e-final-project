## Simple Virtual Memory: Redux

#### Student Perspective

The two key things I learned from this lab was the importance of understanding 
_why_ certain operations have to happen in the order that they do, as well as
why assembly routines need to be used in place of C-code routines in certain 
instances. In previous labs, I (and I imagine many other as well) had a 
tendency to just follow what other people were doing without understanding why
certain things have to happen in a certain order in order for my code to work. 
In the case of virtual memory, the VM system is too finicky to permit such any 
lapses in logic. Working through implementing the assembly routines for the VM
system was a good exercise in understanding why certain things have to happen
in certain places. 

------------------------------------------------------------------------------

#### Key Points from the Lab

##### 1. When to flsush and reset state? 


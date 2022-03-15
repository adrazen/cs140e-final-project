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

##### 1. Importance of Flushing the Cache

As you implemented the assembly routines for this lab, one of the key things
you had to pay attention to was what happens to the cache. According to chapter
`B4: Virtual Memory System Architecture` in the `ARM Architecture Reference Manual` 
(our tried-n-true guide for this course), we know that we need to flush any 
caches that are associated with virtual memory mapping. The exact explanation
on B4-7 states: "Enabling or disabling the MMU effectively changes the 
virtual-to-physical address mapping (unless the transation tables are set up to
implement a flat address mapping). Any virtually tagged caches, for example, that
are enabled at the time need to be flushed." So what exactly does it mean to 
"flush" the caches? And why is it necessary?

Broadly speaking, the term **flushing the cache** refers to cleaning and 
invalidating the contents of the cache. Flushing the cache is required whenever
the contents of external memory have been changed and you want to remove stale
data from the cache. Specifically, enabling and disabling the MMU is one case
where we need to make sure we flush the appropriate caches. For instance, let's
take a look at the first few lines of the implementation of `mmu_disable_set_asm`:

    // void mmu_disable_set_asm(cp15_ctrl_reg1_t c);
    MK_FN(mmu_disable_set_asm)

      @ need to flush any virtually-tagged caches  
      mov r2, #0
      INV_ICACHE(r2)
      
      mov r2, #0
      CLEAN_INV_DCACHE(r2)

      @ use a Data Synchronization Barrier (DSB)
      mov r2, #0
      DSB(r2)
      
      ... remainder of implementation omitted ...


So what exactly are `INV_ICACHE` and `CLEAN_INV_DCACHE` doing, and what's
the difference between the two?

Let's start with `INV_ICACHE`, which _invalidates_ the instruction cache 
(also known as the I-cache). Invalidating the cache means clearing the 
cache of all of its data. This is done by clearing the valid bit of 
one or more cache lines. The cache must always be invalidated after reset 
as its contents will be undefined. If you take a look at the implementation
of `INV_ICACHE` inside of `armv6-coprocesser-asm.h`, you will see how we 
can invalidate the entire I-cache. 

In addition to just invalidating the cache, we can also invalidate _and clean_
the cache. This is what we do in `CLEAN_INV_DCACHE`, where we invalidate and 
clean the data cache (also known as the D-cache). When we clean the cache, 
we are writing the contents of dirty cache lines out to main memory and clearing
the dirty bit(s) in the cache line. This makes the contents of the cache line 
and main memory coherent with each other. 

Given these differences, this means that when we "flush" the cache, we can 
either: clean the cache, invalidate the cache, or clean _and_ invalidate 
the cache. It's important to know which of these three options we need to 
use in order to make sure that memory code gets stored correctly. 

Finally, after we have flushed the cache (whichever combination we need to use), 
we need to make sure we use a Data Synchronization Barrier (DSB) to ensure 
that all cache maintenance operations are completed. As noted on B2-21 of the
`ARM Architecture Manual`: "DSB causes the completion of all cache maintenance 
operations appearing in program order prior to the DSB operation, and ensures
that all data written back is visible to all (relevant) observers."

##### 2. A Rundown on the Flushes

Another aspect of this lab that you may have found confusing is the plethora
of different types of "flushes" and memory synchronization operations. If you 
were confused about what the differences between these operations are (as we 
were), here's a concise of rundown of what each operation does, and why it's
important: 

- `PREFETCH_FLUSH`: The `PREFETCH_FLUSH` instruction ensures that all 
   instructions that follow the `PREFETCH_FLUSH` are fetchec from cache 
   or memory after the flush has been completed. 

   This is important when you are: changing the ASID, completing
   cache maintenance operations or branch predictor maintenance operations, or 
   changing the CP15 registers. 

- `FLUSH_BTB`: The `FLUSH_BTB` instruction flushes the Branch Target Buffers
   (BTB) to ensure that we flush branch prediction logic. 
   
   This is important when you are: enabling or disabling the MMU, writing new
   mappings to the page tables, or changing the TTBR0, TTBR1, or TTBCR. 

- `DSB`: The `DSB` instruction is a Data Synchronization Barrier (DSB) that acts
   as a special type of memory barrier. The `DSB` instruction is useful becuase
   it forces a "waiting period" before given new set of instructions can execute. 
   In a way, you can think of the `DSB` instruction as preventing race conditions. 
   The `DSB` instruction ensures that no instructions that come after the `DSB` 
   can execute until _all_ of the instructions before the `DSB` have finished 
   exeucting. 
   
   This is important when you are: completing cache maintence operations, or 
   completing branch predictor maintenance operations. 

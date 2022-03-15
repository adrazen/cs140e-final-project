## Simple Virtual Memory: Redux

#### Student Perspective

The key thing I learned from this lab was how a virtual memory system is 
_actually_ designed, and how these design specifications are converted to code. 
The key idea behind a virtual memory system is fairly intuitive (coverting 
physical addresses into virtual addresses, and vice versa) but the actual 
implementation of such a system requires a little extra thinking. 

------------------------------------------------------------------------------

#### Key Points from the Lab

##### 1. Importance of Error Checking

When dealing with anything memory-related, it is increasingly important to do 
the appropriate error-checking to make sure that the memory addresses that we 
are dealing with actually makes sense. For instance, when mapping a section from
a virtual address to a physical address, we need to do appropriate error checking
to make sure each component of the mapping is valid. Here is the implementation 
of `mmu_map_section` to see this in play:

    fld_t * mmu_map_section(fld_t *pt, uint32_t va, uint32_t pa, uint32_t dom) {
      // make sure that the domain is valid given we only have 16 domains
      assert(dom < 16);
      // make sure that both physical address and virtual address is valid, 
      // because everything should be in units of 1MB sections
      assert(mod_pow2(va, 20));
      assert(mod_pow2(pa, 20));
    }


##### 2. Dealing with Memory Errors (and How to Implement Error Handling More Generally)

Another key aspect of this lab is ensuring that memory errors are reported to the 
OS in a timely fashion so that they can be dealt with immediately, and correctly. 
This model of error handling is important because it can be replicated in other 
aspects of an operating system and its implementation proves useful when thinking 
about how to deal with error handling more generally in an operating system context. 

To summarize, the implementation of error handling begins with setting the address 
of the interrupt table so that the code knows where to go whenever an interrupt occurs.
Here's how we do 
it in `mmu_install_handlers` in `mmu.c`:

     void mmu_install_handlers(void) {
         extern uint32_t _interrupt_table[];
         vector_base_set(&_interrupt_table);
     }


Next, we need to install an exception handler that can be referenced whenever an error 
occurs. This ensures that whenever some sort of fault occurs, the interrupt table knows 
where to jump to, and that there is a routine that is defined to define the "error 
handling" behavior for that fault. In the case of memory errors, these are considered 
_data_ errors, which means we will branch to the `data_abort_asm`, which will then execute
the `data_abort_vector`. You can take a look at `interrupts_asm.S` in `libpi/src` to 
refresh on this control flow. 

Ultimately, this means that we need to define a `data_abort_vector` function that will 
define the specific behavior for when such a data access error occurs. In our case, we 
do this in `mmu-except.c` where we define a `data_abort_vector` function that will deal
with a data abort fault appropriately. In our case, we do a combination of trying to 
remedy the error in case the error is "fixable" as well as simply alerting of an error 
in the case the error is "unfixable". Here is one possible implementation of the 
`data_abort_vector` function in `mmu-except.c`:

    void data_abort_vector(unsigned lr) {
      // use Data Fault Status Register to get cause of the fault
      unsigned dfsr_value;                                              
      asm volatile ("MRC p15, 0, %0, c5, c0, 0" : "=r"(dfsr_value)); 
      unsigned dfsr_bits = bits_get(dfsr_value, 0, 3);

      // check that the fault is indeed a section fault
      if(bit_is_on(dfsr_value, 10)){ 
        panic("should get section violation fault!\n"); 
      }

      // use Combined Data/FAR to get the fault address
      unsigned far_value;                                              
      asm volatile ("MRC p15, 0, %0, c6, c0, 0" : "=r"(far_value));

      // check faulting address to ensure it is reasonable and makes sense
      assert(far_value == proc.fault_addr || far_value == proc.die_addr);

      // round faulting address to nearest multiple of 2^20 (1MB) for mapping purposes
      unsigned map_fault_address = ((far_value >> 20) << 20); 
      // error is "fixable" because it occured within 1MB of the stack pointer
      if ((map_fault_address < proc.sp_lowest_addr) && (map_fault_address > (proc.sp_lowest_addr - OneMB))) {
        // cause of fault is translation error
        if (dfsr_bits == 0b0101) {
          mmu_map_section(proc.pt, map_fault_address, map_fault_address, proc.dom_id); 
        }
        // cause of fault is permissions error: change permissions
        if (dfsr_bits == 0b1101) {
          fld_t *fte = mmu_lookup_section(proc.pt, proc.fault_addr);
          proc.pt->AP = AP_rw;
        }
        mmu_sync_pte_mods();
      } else {
        // translation error that is out of bounds
        if (dfsr_bits == 0b0101) {
          panic("attempting to store unmapped addr\n");
        }
        // tranlsation error with incorrect permissions
        if (dfsr_bits == 0b1101) {
          panic("attempting to store addr with permission error\n");
        }
      }
    }

## UART: Redux

#### Student Perspective

I remember being in the middle of this lab, and having an a-ha moment as I 
realized just how useful the pre-reading and questions to check my understanding
actually were. Coming into lab with at some level of understanding for what needed
to be translated into code was a great way to jumpstart the lab. I also noticed
a big difference between this lab and `gpio`; having just a single
lab's worth of exposure to the Broadcom doc under my belt made this lab feel
like I was really getting my feet under me. Also, the challenge of writing a device's
`_init` function was an exciting step forward.

------------------------------------------------------------------------------

#### Key Points from the Lab

##### 1. Importance of device memory barriers (i.e. `dev_barrier()`)
Your `uart_init` function is no small piece of code. At this point, we are working 
with multiple devices in a single function, accessing various memory spaces. For 
example, you set the alternate function on the GPIO tx/rx pins by writing to the GPFSELn
register (1st peripheral device: GPIO), then on the Pi (2nd device) itself, you accessed the `AUXENB`
register to enable another peripheral, the mini-UART (3rd device). Between all of these various
memory access steps, it's important to set up **memory barriers**, ensuring that instructions
run to completion and avoiding _unpredictable_ behaviors. This is why you needed to use
`dev_barrier()` between the various device access instructions. It's also a safe call to place
a barrier at the beginning and end of your other `uart_` functions, since you can't guarantee
where these functions are being called.

##### 2. The "read-modify-write" pattern (RMW)
This phrase has probably been mentioned before/during this lab. At it's core, this is simply
a method of preserving register values rather than clobbering them by doing a blunt write. 
Let's look at the `AUXENB` Register as an example. The Broadcom doc notes that this register 
"is used to enable the three modules; UART, SPI1, SPI2." Toggling one of the lower three
bits will have a very large effect on one of these modules' states. So how do we turn enable our
UART while ensuring we don't mess with the bits corresponding to the SPI modules? RMW!

First, we read in the entire `AUXENB` register value. Then, we set the relevant UART bit (and ONLY the relevant bit) of that value.
Then, we write the value back to the register. Read, modify, write! It's a crucial pattern, and learning
when/when not to use it will come in very handy over these next few weeks!



##### 3. Common errors and misconceptions
- Outputting `"hellhellhellhell..."`:  While attempting to successfully run `1-uart/hello`, 
a good number of folks ran into this behavior. As this is a looping behavior related to I/O, 
an immediate alarm bell that you'd want to go off is "Am I flushing data correctly?". In this case,
at times people were either not flushing in their `uart_disable()` (necessary! we want to pause 
until all the data in the queue has been sent on the wire + dealt with/thrown away), 
or some piece of their flush functionality (e.g. `uart_tx_is_empty()`) was ineffective.

- Incorrect register addresses: When working with datasheets as verbose as the Broadcom, it can be
difficult at times to avoid tiny errors when transferring information between the document
and your code. However, when an error ends up in an important variable (e.g. a register address),
it can sometimes lead to hard-to-debug behaviors. A good method for both this lab and those following
is to continually check important "magic" numbers with your peers and triple check against the docs!
Sometimes changing a single digit can be the answer to your problems.

- `PUT/GET8` vs `PUT/GET32`: We are working directly with device memory in this lab. Regardless of 
whether you're working with a single byte (which perhaps PUT/GET8 seems built for), you cannot use 
the `PUT/GET8` functions here. ARM CPU registers (`AUXENB`, `AUX_MU_LSR_REG`, `AUX_MU_IO_REG`, etc..) are
**_32 bit registers_**. To read/write registers (aka "device memory"), you have to read/write the **_whole_**
register (and perhaps mask the value you've read to single out a byte). `PUT32` and `GET32` are your friends!
As an example, here's how you might return a single byte from the RX queue:

       return (GET32(IO_REG) & 0XFF);

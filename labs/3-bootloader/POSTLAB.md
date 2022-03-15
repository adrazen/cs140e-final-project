## Bootloader: Redux

#### Student Perspective

Welcome to the world of debugging! This lab felt like a very systems-y lab in 
that there was a significant debugging component to getting this lab to work. 
The main thing that I learned in this lab was the importance of having a good
debugging approach to figure out what my bugs were. Today's redux takes the 
form of a rundown of common issues that people may have faced, including 
explanations of why these issues arose. 

------------------------------------------------------------------------------

#### Key Points from the Lab

##### 1. What's `kernel.img`? 

When replacing the staff bootloader with your own bootloader, you had to replace
the `kernel.img` file on your SD card. It's easy to simply blindly follow these
instructions without understanding exactly what you are doing, so here's an 
explanation of the _why_. In the operating systems world, an **image** is a binary 
form of an operating system that can be loaded onto a device. In the embedded world, 
images are eVeRyThInG. 

More speficially, the `kernel.img` file is an image file representing the Linux 
kernel (hence the name). You can download the `kernel.img` file onto the pi in order 
to boot the pi into becoming an operating system. Neat!

##### 2. Heap vs Stack Allocation

In many of the previous courses you've taken, you've likley always gotten explicit 
instructions about whether a given piece of data should be allocated on the heap 
or the stack. When implementing the `read_file` function inside of `read_file.c` for 
this lab's prelab, you may have run into issues if you tried returning a stack-allocated
string instead of a heap-allocated one. For instance, take a look at the difference 
between these two implementations. 

To begin, here is an implementation that uses stack-allocation: 

    void *read_file(unsigned *size, const char *name) {
      ... implementation details for reading file into a stat object ommitted ... 
      char buf[size];
      int fd = open(name, O_RDONLY);
      read(fd, buf, file_info.st_size);
      close(fd);
      return buf;
    }

Now consider this implementation that uses heap-allocation: 

    void *read_file(unsigned *size, const char *name) {
      ... implementation details for reading file into a stat object ommitted ... 
      char *buf = calloc(size); 
      int fd = open(name, O_RDONLY);
      read(fd, buf, file_info.st_size);
      close(fd);
      return buf;
    }
    
At first, the difference between the two implementations may seem subtle (if 
not trivial). However, the difference between the two types of allocations is 
important and can lead to some nasty and hard to track down bugs. In fact, some 
implementations will work with stack allocation when doing initial testing (such 
as running the `read-test` in `prelab-tests`) but will only start causining 
problems when they are run in the full bootloader program itself. For those of you 
who ran into this issue, you will likely have run into a not-so-pleasent error: a 
SEGFAULT. There are a few possible issues with using stack allocation instead of 
heap allocation. 

The main issue with a stack allocated string is that it may go out of scope. 
Variables created on the stack will go out of scope and are automatically deallocated.
Meanwhile, variables on the heap must be destroyed manually and never go out of scope.

##### 3. Empty Strings for Days

Many of you likely ran into issues with empty strings echoing, even after the 
communication between the Unix side and the Pi side was done. There are a number 
of different causes for this behavior but here are a few. 

One common issue stems back to, you guessed it, your `read_file` implementation in
the prelab. If you read the file incorrectly (perhaps by trying to read _too_ much
of the file), then you may have seen empty strings echoing on your screen. In order
to make sure you are reading the correct amount of data, you'll want to double 
check your call to `read()` inside of `read_file`: 

  read(fd, buf, file_info.st_size);

Specifically, you want to make sure you are only reading as many bytes as are in the
file itself. This is important becuase the buffer you declare may be larger than the
file itself (given that we declare the size of the buffer to be a roudned up version 
of the size of the file.)

##### 4. Following a Protocol: Write, Read, and Check

One of the most exciting part of today's lab was also one of the most challenging
parts of today's lab: we were able to get two separate programs to talk to one another! 
In order to make this happen, we defined a communication protocol between the Pi side
and the Unix side. This means that whenever the Pi sends data to the Unix side, the Unix
side needs to be ready to recieve that data and process it correctly. When the Unix side
responds back to the Pi side, the Pi side needs to be ready to recieve that response and
process it correctly. Following this protocol means that both sides need to accept data
from the other side (even if they aren't interested in the data that the other side 
sends them) and they need to send data only when the other side is ready and willing to
listen. In other words, each side needs to patiently wait its turn. 

When implementing such a communication protocol, it's good practice to outline the order
of communication, and add error checking whenever a communication happens. Let's take a 
look at how we do this here. 

First, we define the protocol. This means that whenever one of the two communicating sides
wants to send data, it needs to make sure that it is indeed its turn to send data and to 
make sure that it is sending data in the right order. We define the bootloader protocol 
using the following diagram: 

     =======================================================
             pi side             |               unix side
     -------------------------------------------------------
      put_uint(GET_PROG_INFO)+ ----->

                                      put_uint(PUT_PROG_INFO);
                                      put_uint(ARMBASE);
                                      put_uint(nbytes);
                             <------- put_uint(crc32(code));

      put_uint32(GET_CODE)
      put_uint32(crc32)      ------->
                                      <check crc = the crc value sent>
                                      put_uint(PUT_CODE);
                                      foreach b in code
                                           put_byte(b);
                              <-------
     <copy code to addr>
     <check code crc32>
     put_uint(BOOT_SUCCESS)
                              ------->
                                       <done!>
                                       start echoing any pi  
                                       ouput to the terminal.
     =======================================================
     
Second, when ensure that we read data appropriately. In the case of the Unix side, it 
uses the `get_op()` function to read a single 4-byte word from the Unix side. Meanwhile, 
the Pi side uses `boot_get32()` to read a single word from the Pi side. 

The final step, is to verify that data we recieved is indeed what we expect. Whenever
we define a communication protocol, we should not trust _anything_ the other side does. 
This means that we need to verify everything that the other side does and ensure that 
they are sending us data in the right order and that the data they are sending us is
correct. In the context of this lab, the Unix side uses the `ck_eq32()` function to verify 
the data that is sent by the Pi, and the Pi uses the `crc32()` to calculate checksums 
on any data that it sent. 

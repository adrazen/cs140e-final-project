## Lab: write your own UART implementation.

### Introduction

By the end of this lab, you'll have written your own device driver for
the pi's mini-UART hardware, which is what communicates with the TTY-USB
device you plug into your laptop.  This driver is the last "major" piece
of the pi runtime that you have not built yourself.  At this point, ideally you can look at each line of code in the system, and should
know why it is there and what it is doing.  Importantly you have
reached that magic point of understanding where: there is nothing else.

#### Understanding the UART (extremely cursory)
UART - Universal Asynchronous Receiver/Transmitter - is a serial communication
protocol (protocol for sending bits of data serially, one at a time). Typically,
this protocol sends one byte of data at a time, and packages it in a frame structure (see diagram below).
Recall, from last lab, that we noted that a program could be sent from your laptop
to the pi side of the bootloader via the UART. The UART handles serial packets of data traveling in both directions (receive/transmit) between the pi and laptop via the TTYUSB.

Common frame structure:
- START bit: Always low. Indicates serial communication has begun.
- Data bits packet: Always follows start bit. Typically 8-bit data packet, but can range.
- STOP bit: Always follows data packet, high. Indicates end of frame.

<img width="500" alt="image" src="https://user-images.githubusercontent.com/40475205/158321484-ba60d857-0bab-49c3-9fa4-4e607551e6d5.png">

The pi uses a mini-UART which is mapped to two gpio pins for tx/rx (transmit/receive).
Today you'll write code to initialize this UART in the correct configuration (frame size, 
[baud rate](https://en.wikipedia.org/wiki/Baud)), as well
as read received data, and write data to transmit!

This lab will build off of the experience you gained in the `gpio` lab
while working with the Broadcom doc. We will once again be leaning
heavily on that doc `docs/BCM2835-ARM-Peripherals.annot.PDF`,
and your newfound skills in parsing useful info
from this type of hardware datasheet will be boundlessly useful here!

*NOTE*:
  - Make sure you read through the [mini-UART cheatsheet](miniUART.md)

In addition, you'll implement your own *software* UART implementation
that can entirely bypass the pi hardware and talk directly to the
tty-USB device itself.  I.e., without using the driver above, but instead
building an 8n1 serial protocol with the CP2102 device over two GPIO pins.
Multiple reasons:

   1. The most practical reason: this gives you a "console"
      `printk` you can use when the UART is occupied, which turns out to
      be surprisingly useful.  For example, when you write the UART driver
      above, you can't actually use `printk` to debug it, since doing so
      requires a UART!  As another, when we use the ESP8266 networking
      device in a couple of labs: it needs a UART to communicate with
      the pi.  Without a second UART, we would have no way to print to
      the laptop.  Again: hard to debug if you have no idea what the pi
      is doing.

   2. Gives you an existence proof that you don't need to need to use
      hardware but can roll your own.  This applies to all the different
      hardware protocols  built-in to the pi, such as I2C and SPI.

   3. Writing your own version of a hardware protocol gives you more
      of an intuition for the problems that happen at this level..
      For example, how do you deal with concurrency between two entirely
      separate devices when we don't have the usual fall-back on locks,
      etc?  (Hint: the use of messages and very precise time deadlines
      are fairly common.)

   4. This gives you a small demonstration of how your pi setup,
      which appears trivial, is a hard-real time system, where with
      some care, it possible to hit nanosecond deadlines reliably.
      Difficult to do this on a "real" system such as MacOS or Linux.

### Deliverables

Show that:
   1. `1-uart`: `1-uart/hello` works using your `1-uart/uart.c`.
      Your `uart.c` code should make it clear why you did what you did,
      and supporting reasons --- i.e., have page numbers and partial
      quotes for each thing you did.

       After this replacement, the tracing tests from the last two labs
       --- `3-bootloader/checkoff-tests` and `2-cross-check/2-trace/tests`
       --- should behave identically

   2. `2-bootloader`: Your bootloader works just using your
      `uart.o` and the libpi.

       Again: After this replacement, the tracing tests from
       the last two labs --- `3-bootloader/checkoff-tests` and
       `2-cross-check/2-trace/tests` --- should behave identically

   3. Your fake pi implementatoin for `uart.c` gives the
      same hash as everyone else.  Note, there are multiple ways to do
      same thing, so maybe do the first one as a way to resolve ambiguity.

   4. Your software UART can reliably print and echo text between the pi
      and your laptop.

-------------------------------------------------------------------------
#### The general goals

The main goal of this lab is to try to take the confusing prose and
extract the flow chart for how to enable the miniUART with the baud rate
set to 115200.  You're looking for sentences/rules:

  * What values you write to disable something you don't need
	(e.g., interrupts).

  * What values you have to write to enable things you need (miniUART).

  * How to get the gpio TX, RX pins to do what you need.

  * How to receive/send data.  The mini-UART has a fixed sized transmit
	buffer, so you'll need to figure out if there is room to transmit.
	Also, you'll have to detect when data is not there (for `uart_getc`).

  * In general, anything stating you have to do X before Y.

You don't have enough information to understand all the document.  This is
the normal state of affairs.  So you're going to start practicing how
to find what you need and interpolate the rest.

The main thing is to not get too worked up by not understanding something,
and slide forward to what you do get, and cut down the residue until
you have what you need.

-----------------------------------------------------------------------
### Part 1. implement a UART device driver:

Our general development style will be to write a new piece of
functionality in a private lab directory where it won't mess with anything
else, test or (better) equivalence check it, and then, migrate it into
your main `libpi` library so it can be used by subsequent programs.

Concretely, you will implement the routines in `4-lab/1-uart/uart.c`.
The main tricky ones:

  1. `void uart_init(void)`: called to setup the miniUART.  It should
     set the baud rate to `115,200` and leave the miniUART in its default
     `8n1` configuration.  Before starting, it should explicitly disable
     the UART in case it was already running (since we bootloader,
     it will be).

  2. `int uart_getc(void)`: blocks until it can read a byte from
     the mini-UART, and returns the byte as a signed `int` (for sort-of
     consistency with `getc`).

  3. `void uart_putc(unsigned c)`: puts the byte `c` onto the UART transmit
     queue.  If necessary, it blocks until there is space.

General approach for `uart_init`:
  1. You need to turn on the UART in AUX.  Make sure you
     read-modify-write --- don't kill the SPIm enables.
  2. Immediately disable tx/rx (you don't want to send garbage).
  3. Figure out which registers you can ignore (e.g., IO, p 11).
     Many devices have many registers you can skip.
  4. Find and clear all parts of its state (e.g., FIFO queues) since we
     are not absolutely positive they do not hold garbage.  Disable
     interrupts.
  5. Configure: 115200 Baud, 8 bits, 1 start bit, 1 stop bit.  No flow
     control.
  6. Enable tx/rx.  It should be working!

If you run `make` in `4-uart/1-uart` it will build:
  - A simple `hello` you can use to test. I'd suggest shipping it over with your
    bootloader.  

A possibly-nasty issue: 

  1. If you test using the bootloader (recommended!), that code obviously
    already initializes the UART.  As a result, your code might appear
    to work when it does not.

  2. To get around this issue, when everything seems to work, change
     the `Makefile` to use `hello-disable.c` instead of `hello.c`
     This will repeatedly disable and enable the uart --- not a perfect
     test, but a bit more rigorous.
 
  3. Once your code works, make swap out the `uart.c` in the `libpi/Makefile`
     to use yours:  copy `uart.c` to `src/uart.c`, remove it from the current
     `1-uart/Makefile`, recompile.

  4. Make sure `make check` for the tests in lab 3 still work.
     and see that it boots the `hello` you used above.

-----------------------------------------------------------------------
### Part 2. replace your bootloader.

Now change your bootloader to use the new `uart.c`:
  1. Copy `pi-side/get-code.h` from the last lab into `2-bootloader`.
  2. Change it to print out your name and "lab 4" when done.
  3. Install it on your SD card.
  4. As with the previous step, make sure the tracing tests from the 
     last two labs --- `3-bootloader/checkoff-tests` and
     `2-cross-check/2-trace/tests` --- still behave identically
     (`make check` passes).

-----------------------------------------------------------------------
##### Part 3. `libpi-fake`

Your top level directory now has a `libpi-fake` directory. 

  1. Check that the old code works by running the tests by
     by doing `make check` in `libpi-fake/tests-gpio`:
  2. Check that the uart code works by running the tests in `tests-uart`.
     You'll have to compare your `.out` files with other people.  You
     probably want to run these by hand one at a time.

     NOTE: A big disadvantage of these checks is that they require
     the same reads and writes be done in the same order.  This is
     unrealistic and makes checking kinda of a pain.   We will fix this
     in the homework.
-----------------------------------------------------------------------
### Part 4. implement `sw_putc` for a software UART.

Note:
   - To use our `sw-uart.o`: `staff-objs/sw-uart.o` to the libpi `Makefile`.
   - To use yours: uncomment `sw-uart.o` from `4-sw-uart/Makefile`.

This part of the lab is from a cute hack Jonathan Kula did in last year's
class as part of his final project.  

While the hardware folks in the class likely won't make this
assumption-mistake it's easy as software people (e.g., me) to assume
things such as: "well, the tty-usb device wants to talk to a 8n1-UART
so I need to configure the pi hardware UART to do so."  This belief,
of course, is completely false --- the 8n1 "protocol" is just a fancy
word for setting pins high or low for some amount of time for output,
and reading them similarly for input.  So, of course, we can do this
just using GPIO.  (You can do the same for other protocols as well,
such as I2C, that are built-in to the pi's hardware.)


As described in 
[UART](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter)
the protocol to transmit a byte `B` at a baud rate B is pretty simple.

  1. For a given baud rate, compute how many micro-seconds
     `T` you write each bit.  For example, for 115,200, this is:
     `(1000*1000)/115200 = 8.68`.  (NOTE: we will use cycles rather
     than micro-seconds since that is much easier to make accurate.
     The A+ runs at `700MHz` so that is 700 * 1000 * 1000 cycles per
     second or about `6076` cycles per bit.)

To transmit:
  1. write a 0 (start) for T.
  2. write each bit value in the given byte for T (starting at bit 0, bit 1, ...).
  3. write a 1 (stop) for at-least T.

Adding input is good.  Two issues:
  1. The GPIO pins (obvious) have no buffering, so if you are reading from the RX
     pin when the input arrives it will disappear.
  2. To minimize problems with the edges of the transitions being off
     I'd have your code read until you see a start bit, delay `T/2` and then
     start sampling the data bits so that you are right in the center of 
     the bit transmission.

The code is in `4-uart/4-sw-uart`:

  1. Your software UART code goes in `sw-uart.c`.
  2. You can test it by running `4-sw-uart/hello`.

-----------------------------------------------------------------------------
#### Extension: speed up your bootloader / my-install

Our current `my-install` and bootloader uses a pretty slow baud rate.  Later
in the quarter this can lead to big lags when we send over larger programs.
Start changing your `my-install` and `bootloader` to use a faster baud.
(There will be a limit on how fast your OS can handle.)   You can do some
simple tests to figure out what seems to work easily, maybe back down a 
step, and then rerun the tracing tests.

----------------------------------------------------------------------------
#### extension: sw-uart speed

Once you have a handle on error, it's a fun hack to see how high of a
baud rate you can squeeze out of this code using the above tricks.
In the end I wound up switching from micro-seconds to using the pi's
cycle counters (`cycle-count.h` has the macros to access) and in-lining
the GPIO routines.  (Note that as you go faster your laptop's OS might
become the problem.)

Note:
  - If you switch baud rates on the pi, you'll need to change
    it in the `pi-cat` code too.***
  - Because the counter will overflow frequently you must be careful
    how you compare values.

----------------------------------------------------------------------------
#### extension: sw-uart use a second tty-usb

We have extras we can give out.

Wiring up the software UART is fairly simple: 

  1. You'll need two GPIO pins, one for transmit, one for receive.
     Configure these as GPIO output and inputs respectively.
     To keep things simple, we just re-use the pins from the hardware uart.

     If you get ambitious we can give you a second tty-usb device and
     you can use that!

  2. Connect these pin's to the CP2102 tty-usb device (as with the 
     hardware remember that `TX` connects to `RX` and vice versa).
  3. Connect the tty-usb to your pi's ground.  ***DO NOT CONNECT TO ITS POWER!!***

  4. When you plug it in, the tty-usb should have a light on it and nothing
     should get hot!

Note, testing is a bit more complicated since you'll have two `UART` devices.

  1. When you connect your pi and figure out its `/dev` name; you will
     give this to the `pi-install` code.

  2. Now connect the auxiliary UART, and figure out its `/dev` name.
     You will give this device name to `pi-cat` which will echo everything
     your code emits.
     
     For example, on Linux, the first device I plug in will be `/dev/ttyUSB0`
     and the second `/dev/ttyUSB1`.  So I would bootload by doing:

            my-install /dev/ttyUSB0 hello.bin

     And running `pi-cat` by:

            pi-cat /dev/ttyUSB1

  3. In general, if your code has called reboot, you do not have to pull
     the usb in/out to reset the pi.  Just re-run the bootloader.  (A simple
     hack is to look at the tty-usb device --- if it is blinking regularly
     you know the pi is sending the first bootloader message :)).
  
Note:  our big issue is with error.  At slow rates, this is probably ok.
However, as the overhead of reading time, writing pins, checking for
deadlines gets larger as compared to `T` you can introduce enough noise
so that you get corrupted data.  Possible options:

  - Compute how long to wait for each bit in a way that does not lead to cumulative
      error (do something smarter than waiting for `T`, `2*T`, etc.)

  - The overhead of time is probably the main issue (measure!), so
      you could inline this.  In general, reading time obviously adds
      unseen overhead, so you have to reason about this.

  - You could also switch to cycles.  (our pi runs at `700MHz` or 7 million
      cycles per second.  You should verify 70 cycles is about 1 micro-second!)

  - Unroll any loop and tune the code.

  - Maybe enable caching.

  - Maybe inlining GPIO operations (probably does not matter as long overhead as `< T`)

  - In general: Measure the overhead of reading time, writing a bit,
      etc.  You want to go after the stuff that happens "later" in the bit
      transmit since it has the most issue and the stuff that has a large
      fixed cost (since that will cause the error to increase the most).

  - Could just hand compute an assembly routine that runs for exactly
      the cycles needed (count the jump and return overhead!).  Or,
      better, write a program to generate this code.

  - Part of "systems" as a discipline is actually measuring what effect
      your changes make.  People are notoriously stupid at actually
      predicting where time goes and what will lead to improvements.

      Of course, measuring the actual error is tricky, since the
      measurement introduces error.  One approach is to write a trivial
      oscilloscope program on the pi that will monitor a GPIO pin on
      another pi for some time window (e.g., a millisecond), record at
      what exact cycle count the pin changed value, and print the results
      after. You'll need a partner (or a 2nd pi) but this is actually a
      close-to trivial program and gives a very good measurement of error
      (it's useful in general, as well).


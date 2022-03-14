## ESP8266 Networking: Redux

#### Student Perspective

This lab is really cool because you get to implement networking using your pi!
The key challenge in this lab is making sure you are recognizing the correct
device, and that you are following the relevant protocols in order to be able
to communicate over a networked channel. Specifically, we use `AT` command to 
communicate with the ESP, and we use HTTP to communicate with the browser. 

------------------------------------------------------------------------------

#### Key Points from the Lab

##### 1. Finding the Right Device

The first step to successfully implementing this lab is making sure that your
`find_tty_usb` function works correctly. The first issue many people run into
is knowing which prefix to look for in their `find_tty_usb` function. Here is 
a sample list of some common prefixes that people might need to look for: 


    static const char *ttyusb_prefixes[] = {
         "ttyUSB",   // linux
         "cu.SLAB_USB", // mac os
         "cu.usbserial",
         "tty.usbserial",
         "tty.SLAB_USBtoUART",
         0
    };

Knowing whether you have found the right device boils down to knowing what the 
difference is between a `tty.*` device and `cu.*` device. In Unix and Linux 
environments, each serial communication port has two parts to it, a `tty.*` and
a `cu.*`. The difference between the two is that a TTY device is used to call 
into a device/system, while the CU device (where CU stands for call-up) is used 
to call out of a device/system. As you can imagine, this difference in function
is extremely important for a networked system (such as the one we built in this
lab). 

##### 2. 

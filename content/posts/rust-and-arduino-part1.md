---
title: "Rust, Arduino and Embedded Development as a Beginner: Part 1"
date: 2020-08-16T18:42:00+02:00
---
**Target audience:** experienced developers with (close to) zero knowledge in
embedded development and electronic.

Introduction
------------

I often got interested by Arduino's because of some specific problems they can
solve, like: programming your thermostat for instance, automate your house,
etc... When I first heard of it a friend of mine gave me some hardware so I can
experiment. Unfortunately I never really managed to get into it mainly because
it was not what I imagined. Coming from a "Python Developer" background, doing
C/C++ felt really bad and using an IDE in Java felt like I had no control or no
idea of what I was actually doing.

But ~2 years ago I've discovered Rust and it changed completely my career.
Rust quickly became my favorite programming language and it allowed me to
explore new horizons. The news did reach me when I heard it is coming to
embedded development and I decided recently to give it a real shot. Not because
I needed anymore, but just because I could.

Arduino and embedded development in general are not easy to grasp as you need
general knowledge in electronic engineering: what is a resistor, how to
measure things, what happens if I measure from here to there, how do I even
find the information about what am I supposed to connect where. But on the
other hand it is important to say that Arduino did make everything very easy
and accessible to anyone... (if you don't mind C/C++ and an opinionated IDE.)

Prerequisites
-------------

 *  Patience
 *  Willingness to learn
 *  Understand the struggle will be real and painful

> Maybe, after years of successfully shipping code, you don't have quite the
> same curiosity, the same candor and willingness to feel "lost" that you did
> back when you started.
>
> -- ["Frustrated? It's not you, it's Rust", by Amosi Wenger](https://fasterthanli.me/articles/frustrated-its-not-you-its-rust)

### Help needed?

It is a good habit to search the web the answers to your questions. I honestly
find a great satisfaction when I find and learn things all by myself but it is
sometimes harder, frustrating and time-eater. In any case, at some point you
**will need** help and I would suggest to
[join the rust-embedded discussion channel](https://matrix.to/#/#rust-embedded:matrix.org)
and check the content of the
[embedded Rust documentation](https://docs.rust-embedded.org/).

### My own hardware choice

 *  [SparkFun Qwiic Pro Micro - USB-C (ATmega32U4)](https://www.sparkfun.com/products/15795)
 *  [Zio Qwiic OLED Display (1.5inch, 128x128)](https://www.sparkfun.com/products/15890)
 *  [Qwiic Cable - 50mm](https://www.sparkfun.com/products/14426)
 *  [USB to TTL Serial Cable](https://www.sparkfun.com/products/12977)

Getting started vewy qwiickly
-----------------------------

If like me you don't want to spend too much time on the hardware and start the
actual code. You can get a board with a "Qwiic" connector. This is some kind of
universal connector with a special circuitry that will handle any voltage
adjustment for you. The only thing you need are devices with Qwiic connectors.
They can be connected in serial so you can actually connect multiple of them.

Now to get a bit more into the details of what you need to know. This is an I2C
connector. In the world of embedded development you may encounter 3 different
kinds of inter-device communication: UART, SPI and I2C. I'll drop
[a link](https://www.engineersgarage.com/tutorials/understanding-the-i2c-protocol/)
to a very good documentation of I2C and its relation to the others. I'd
suggest to read at least the beginning  to understand the differences between
the 3 and how I2C works.

Choosing the hardware
---------------------

Ideally you want a board for which we already have support in Rust. That is why
I took a board with an ATmega32U4 processor. This is the same microchip that
you can find in the Arduino Leonardo so the assembly instructions will be
identical (even the pins are identical). It is also worth noting that this
microchip is part of a family named
[AVR microcontrollers](https://en.wikipedia.org/wiki/AVR_microcontrollers).

My project consisted only of a board (also known as MCU! You need to know that)
and a screen (any screen). All I wanted to do is to display an animation on
this screen. So any device that has Qwiic connector would have work for my
project.

First step: making sure everything works
----------------------------------------

The good thing with Arduino is that there are already examples for everything.
For example the screen I bought had a tutorial for Arduino on how to display
simple text on the screen. You just connect all the things, follow the
tutorial, copy the example, click the button to compile & upload to the device
and it should work immediately. I guess this part depends a bit where you
bought your hardware but if it is Arduino or if you bought on SparkFun, you
will get something that works out of the box.

I assume you can buy cheaper boards, copies, etc... that might not have the
exact same characteristics than the original hardware and it might not work as
easy as you might want. If like me you are starting, you should probably avoid
that.

Starting with Rust
------------------

Now that we have validated that our hardware is working we can validate that we
can compile and run Rust code on the thingy. For that you will need to
[glone](https://github.com/cecton/cecile/blob/master/bin/git-glone) the sources
of [avr-hal](https://github.com/Rahix/avr-hal).

Follow the README instructions to make a LED blink and enjoy the learning
journey.

### Troubleshooting

1.  *Oh hey, I followed the instructions and now I have a hex file. What do I
    do?*

    During my first experience with the Arduino I did learn something: how to
    flash a board without the IDE. Of course I have entirely forgot it so I
    looked for "arduino leonardo flash board command line". I did find the
    command but I strongly suggest to not do that and hack the Arduino IDE
    instead.

    The configuration files of Arduino are located somewhere around
    `~/.arduino15` on a Linux system.

    Once you find the source, start looking for the `avrdude` binary. This
    program allows you to (I assume) flash all the AVR based board but it will
    require some parameters for your specific board.

    ```text
    > find . -name avrdude -type f
    ./packages/SparkFun/hardware/avr/1.1.13/bootloaders/optiboot/avrdude
    ./packages/arduino/tools/avrdude/6.3.0-arduino17/bin/avrdude
    ```

    One of the result is used by Arduino to upload your code. Lets try to
    figure out which one.

    ```text
    > ls -l $(find . -name avrdude -type f)
    .rw-r--r--   0 cecile  5 Nov  2019 ./packages/SparkFun/hardware/avr/1.1.13/bootloaders/optiboot/avrdude
    .rwxr-xr-x 281 cecile 14 Aug 15:52 ./packages/arduino/tools/avrdude/6.3.0-arduino17/bin/avrdude
    ```

    We can exclude the 0 bytes one, which leaves us with the correct `avrdude`
    binary. Let's see what commands it runs. To do that we are going to wrap
    the executable in a bash script that will log all the commands that have
    been run:

    ```text
    mv ./packages/arduino/tools/avrdude/6.3.0-arduino17/bin/avrdude \
        ./packages/arduino/tools/avrdude/6.3.0-arduino17/bin/avrdude-original
    ```

    Now make a new file
    `./packages/arduino/tools/avrdude/6.3.0-arduino17/bin/avrdude` and put
    this:

    ```bash
    #!/bin/bash

    echo "$@" >> /tmp/avrdude.log

    /home/cecile/.arduino15/packages/arduino/tools/avrdude/6.3.0-arduino17/bin/avrdude-original "$@"
    ```

    **Note:** try to use the full path to the original executable to make sure
    that if the script is ran from a different directory it will still work.

    Make the file executable:

    ```
    chmod +x ./packages/arduino/tools/avrdude/6.3.0-arduino17/bin/avrdude
    ```

    Now click on the upload button of the Arduino IDE again and see what has
    been saved in `/tmp/avrdude.log`:

    (The following output is from my machine. You must check your output.)

    ```text
    -C/home/cecile/.arduino15/packages/arduino/tools/avrdude/6.3.0-arduino17/etc/avrdude.conf -q -q -patmega32u4 -cavr109 -P/dev/ttyACM0 -b57600 -D -Uflash:w:/tmp/arduino_build_808706/Blink.ino.hex:i
    ```

    These are the arguments passed to avrdude to flash your board. So your own
    `avrdude` command to flash with your Rust program should look like this:

    ```text
    ~/packages/arduino/tools/avrdude/6.3.0-arduino17/bin/avrdude \
        -C/home/cecile/.arduino15/packages/arduino/tools/avrdude/6.3.0-arduino17/etc/avrdude.conf
        -q -q -patmega32u4 -cavr109 -P/dev/ttyACM0 -b57600 \
        -D -Uflash:w:/tmp/arduino_build_808706/Blink.ino.hex:i
    ```

    The important part of this command are:

     -  `-patmega32u4`: this is my microcontroller
     -  `-cavr109`: this`: this is the "programmer" (I know what is going to be
        your next question!)
     -  `-P/dev/ttyACM0`: that's a serial port to your device. When your device
        is connected, this file (device) appears in `/dev`
     -  `-b57600`: that's the speed of the serial communication. It needs to
        match the program it tries to communicate otherwise they can't
        understand each other. It's a bit like you need to be on the same radio
        frequency if you talk through a radio (maybe a terrible analogy).
     - `-Uflash:w:/tmp/arduino_build_808706/Blink.ino.hex:i`: the `.hex` file
       is in there. That's how you pass the code.

2.  *What is a "programmer"?*

    There is no such thing as asking too many questions. For me (and maybe for
    you) this journey was a real discovery with a lot of vocabulary and basic
    knowledge I miss because I never did electronic and such low level
    programmation. I believe you will get an easier time if you do ask yourself
    every question every time you encounter something new. Hopefully you will
    find most of the answers in this article.

    Boards are almost identical if you have the same microchip. But there is a
    slight difference on how they boot your code. It's similar to having a
    different motherboard on a PC if you prefer, the boot program is provided
    by the board. A "programmer" is a program that will communicate with the
    board and allow flashing your code. You need to use the right programmer
    for your board as the protocol may vary. (Note that a programmer is also a
    piece of hardware capable to flash the board in some cases. Hopefully
    you won't need that.)

3.  *Ok so I copied the command and I get this error. I can't even upload
    Arduino's own `.hex`:*

    ```text
    Connecting to programmer: .avrdude: butterfly_recv(): programmer is not responding

    avrdude: butterfly_recv(): programmer is not responding
    avrdude: error: buffered memory access not supported. Maybe it isn't
    a butterfly/AVR109 but a AVR910 device?
         Double check connections and try again, or use -F to override
    ```

    Don't do `-F`. I have no idea what will happen. But I know it is possible
    to [brick](https://en.wikipedia.org/wiki/Brick_(electronics)) your board so
    it's best to avoid mistakes as much as possible. Check the manual of
    avrdude for more information `man avrdude` (also available online
    [here](https://www.nongnu.org/avrdude/user-manual/avrdude_4.html)).

    You get this error when the board is not in "flashing mode". In other words
    you need to do something first (usually --but not always-- long pressing
    the reset button on the board) before being able to communicate with the
    programmer.  Remember that the programmer on the board is not running all
    the time, it is running only when asked.

    If like me you have one of those recent boards that don't need to press any
    button at all and Arduino magically flash the board, you will need to check
    the documentation provided with your board. If you can't find any useful
    information, try to search on the web, ask someone or contact simply the
    shop where you bought it.

    In my case the procedure was not to long press the reset button but to
    press twice the reset button. I found that on
    [a doc](https://cdn.sparkfun.com/datasheets/Dev/Arduino/Boards/32U4Note.pdf)
    on the website of the shop, in the "Documents" section. When it's done,
    you must run the `avrdude` command immediately. You will see that it is
    possible that the serial device has changed (`/dev/ttyACM0` to
    `/dev/ttyACM1`). That's because your board has rebooted.

    I'm sure at this point you are wondering why the Arduino can flash it
    without pressing the button and how. My guess is that when you compile a
    program with the Arduino IDE a lot of libs are hidden but included in your
    code during the compilation. They seem to open a serial port through the
    USB that allows the device to go into "flash mode". I did try to spy on the
    serial communication using
    [slsnif](https://sourceforge.net/projects/slsnif/reviews/) and there is a
    sequence of bytes sent to the device. I didn't manage to reproduce this
    sequence properly and make it go into flash mode.

    **Important:** please note that your Rust code does not include this reboot
    code. When your Rust code is running, `/dev/ttyACM0` won't be available and
    you won't be able to just press the upload button in the Arduino IDE
    anymore. You will always need to press twice (or long press, whatever it
    is) the reset button when you flash and a Rust code is installed. If you do
    flash with an Arduino code, this will effectively restore that feature (but
    it will work only while the Arduino code is installed, it will be gone if
    you flash a Rust program again).

### Expected status

Right now you should have managed to upload the blink program of `avr-hal`'s
examples and your board should be blinking. Hopefully this first tutorial
managed to get you on track for what will come next.

You can now adventure to the [part 2]({{< ref "rust-and-arduino-part2" >}}),
and learn how to actually draw pixels on the screen.

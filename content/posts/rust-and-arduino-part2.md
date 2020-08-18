---
title: "Rust, Arduino and Embedded Development as a Beginner: Part 2"
date: 2020-08-18T19:49:00+02:00
---
Introduction
------------

After reading the [part 1]({{< ref "rust-and-arduino-part1" >}}), our
adventurers decided to go to the part 2 to see how the hell she managed to
display pixels on an OLED screen.

Serial communication
--------------------

We have seen in the previous part (of the adventure) how to connect and flash
the hardware. Now let's see how to log things with serial communication. That
would become handy if we want to debug.

On the repository of
[avr-hal](https://github.com/Rahix/avr-hal/tree/master/boards/arduino-leonardo/examples)
that you already gloned, you will find that there is also am example for serial
communication. Compile the example, flash the board and try.

### Troubleshooting

1.  Oh, already?

    Communication on serial is really easy and the example works almost
    out-of-the-box so there isn't much to say except solving the issue you will
    encounter.

2.  How do I listen to the serial port?

    Some people use the good old `screen` (a terminal multiplexer also capable
    to connect to a serial port) but I prefered to use `serial-monitor`, a tool
    entirely made with Rust available on
    [crates.io](https://crates.io/crates/serial-monitor).

    The installation is easy with cargo:

    ```
    cargo install serial-monitor
    ```

    This will download the sources, compile and install the binary in
    `~/.cargo/bin` which you can to your `PATH` if it is not already done.

    Now you can connect to the port with the following command:

    ```
    serial-monitor -p /dev/<some-tty> -b <bauds>
    ```

    You need to replace the `/dev/<some-tty>` with the path of your serial
    device. "bauds" are the speed of the serial communication. It needs to
    match the program it tries to communicate otherwise they can't understand
    each other. It's a bit like you need to be on the same radio frequency if
    you talk through a radio (maybe a terrible analogy). (Deja vu?)

3.  I can't find `/dev/ttyACM0` anymore!

    Just like me you thought that since you flashed the device on
    `/dev/ttyACM0` and since there was a serial device when the Arduino program
    was running, you thought that we would have the serial through the USB.

    Well, no.

    People who does embedded development knows it very well, you need to
    connect to the serial interface on the board. Don't look for some kind of
    slot, it's not a slot, it's just some of those pins.

    You can use the "USB to TTL Serial Cable". Connect first to the pin on the
    board and then plug the USB. The pins on your board should be labeled.
    Otherwise, check the documentation.

    I'm sure at this point you wonder how the hell do you connect a serial
    cable to the board. There are 4 colors: red (seems universally for power),
    black (seems universally for the ground), yellow/white (RXD), orange/green
    (TXD). The serial work by **NOT CONNECTING THE RED CABLE AT ALL**, then
    connect the ground to a GND pin, the RXD pin of the client to the TXD of
    the host and the TXD pin of the client to the RXD of the host. (In other
    words you need to connect the yellow/white cable to the TXD pin on the
    board and the orange/green cable to the RXD on the board.)

    The red cable is only used when you need to power the device. If you
    connect it and also connect the USB cable, I believe you will send 10V to
    the board and it may die.

    TXD means "transmit data" while RXD means "receive data".

    Once all 3 cables are connected (GND, TXD, RXD) you can connect the USB to
    your computer. Note that nothing terrible will happen if a cable gets
    disconnected, you can just re-connect it even if the USB is already
    connected. The the red one is more risky though.

    Now that the USB serial cable is connected you should have a new interface
    `/dev/ttyUSB0` (this may vary depending on your system and distribution).
    You can communicate with it using:

    ```
    serial-monitor -p /dev/ttyUSB1 -b 57600
    ```

    At this point you should see something on the terminal or press a key to
    make it react (the example code waits for a key).

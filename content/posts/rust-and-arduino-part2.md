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
    out-of-the-box so there isn't much to say except solving the issue you may
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
    connect it and also connect the USB cable, bad things will happen. You may
    damage one of the device due to overcurrent conditions.

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

Drawing pixels on the screen
----------------------------

### Ensure the screen is detected

Open the i2cdetect example and run it. The USB serial cable must be connected
before the board boots. You should see something like this on the terminal:

```
Write direction test:
-    0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:       -- -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- 3c -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --

Read direction test:
-    0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:       -- -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- 3c
```

If you see something like this instead it means the screen is not detected
(check that it is properly connected).

```
Write direction test:
-    0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:       -- -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --

Read direction test:
-    0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:       -- -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

It is also possible that the cables of the serial moves a bit. Give it a few
tries.

### Ping the screen

If you take the example of blinking a led and mix it with the example of using
i2c, you can make a led that turns on when the screen is connected. In the API
of avr-hal you will find the function
[`ping_slave`](https://rahix.github.io/avr-hal/atmega32u4_hal/i2c/struct.I2c.html#method.ping_slave)
which returns a boolean if the slave answers to the ping. You will need the
address of the device which you can find somewhere in the documentation of the
screen. For the one we have chosen, this address is `0111100`. All the units of
this screen share the same address, it is not unique.

```rust
#![no_std]
#![no_main]

extern crate panic_halt;
use arduino_leonardo::prelude::*;

#[arduino_leonardo::entry]
fn main() -> ! {
    let dp = arduino_leonardo::Peripherals::take().unwrap();

    let mut delay = arduino_leonardo::Delay::new();
    let mut pins = arduino_leonardo::Pins::new(dp.PORTB, dp.PORTC, dp.PORTD, dp.PORTE);
    let mut led_rx = pins.led_rx.into_output(&mut pins.ddr);
    let mut serial = arduino_leonardo::Serial::new(
        dp.USART1,
        pins.d0,
        pins.d1.into_output(&mut pins.ddr),
        57600,
    );
    let mut i2c = arduino_leonardo::I2c::new(
        dp.TWI,
        pins.d2.into_pull_up_input(&mut pins.ddr),
        pins.d3.into_pull_up_input(&mut pins.ddr),
        50000,
    );

    let address = 0b0111100; // replace this by the address of your device

    loop {
        match i2c.ping_slave(address, arduino_leonardo::hal::i2c::Direction::Write) {
            Ok(true) => led_rx.set_low().void_unwrap(),
            Ok(false) => led_rx.set_high().void_unwrap(),
            Err(err) => ufmt::uwriteln!(&mut serial, "Error: {:?}", err).void_unwrap(),
        }
        delay.delay_ms(1000u16);
    }
}
```

You can connect and disconnect the screen while it is running and you will see
the led turning on and off (with a delay of max 1 second).

### Turning on the screen

To turn on the screen we will need to send a command to the screen. If you
[check the documentation](https://cdn.sparkfun.com/assets/1/a/5/d/4/DS-15890-Zio_OLED.pdf)
of our screen you will see that this screen has 2 modes of communication:

 *  one for data: you send the pixel's color with it
 *  one for commands: you can turn on, turn off, stand by, do some scrolling,
    and even do some effects on the image

All the commands are listed in the documentation. The one we are interested in
is the command `AFh`. `h` stands for hexadecimal. This is the command `0xaf` in
our code.

To send the command `0xaf` you will need to prefix it with the byte that will
the screen that this is a command and not a pixel. This is documented in the
I2C protocol section of the documentation but since avr-hal does already all
the work for sending I2C commands, we only need to care about the content of
the packet.

> After the transmission of the slave address, either the control byte or the
> data byte may be sent across the SDA. A control byte mainly consists of Co
> and D/C# bits following by six “0” ‘s.
>
> a.If the Co bit is set as logic “0”, the transmission of the following
> information will contain data bytes only.
>
> b.The D/C# bit determines the next data byte is acted as a command or a data.
> If the D/C# bit is set to logic “0”, it defines the following data byte as a
> command. If the D/C# bit is set to logic “1”, it defines the following data
> byte as a data which will be stored at the GDDRAM. The GDDRAM column address
> pointer will be increased by one automatically after each data write.

In other words, a full byte is sent but only the 2 first bits are used:

 *  To send a command we will need to set the Co bit to 1 and the D/C# bit to
    0.
 *  To send a data on the other hand we will need to either set the Co bit to 0
    or set the Co bit to 1 and the D/C# bit to 1. (I don't know what is the
    difference.)



## Notes


Voltage will only add if you ground one of the power supplies at the positive output of the other
E.g. V1 connects the positive V to node X and the negative to G.
V2 is connects positive to node Y and negative to X

That means that if you measure G -> Y, it's equal to V1 + v2
And if you do that you need to be careful that the supplies do not share a common ground or it can blow up on you

---
title: "Rust, Arduino and Embedded Development as a Beginner: Part 2"
date: 2020-08-18T19:49:00+02:00
---
Introduction
------------

After reading the [part 1]({{< ref "rust-and-arduino-part1" >}}), our
adventurers decided to go to part 2 to see how the hell they managed to display
pixels on an OLED screen.

Serial communication
--------------------

We have seen in the previous part (of the adventure) how to connect and flash
the hardware. Now let's see how to log things with serial communication. That
would become handy if we want to debug.

On the repository of
[avr-hal](https://github.com/Rahix/avr-hal/tree/master/boards/arduino-leonardo/examples)
that you already "gloned", you will find that there is also an example for
serial communication. Compile the example, flash the board and try.

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

    ```text
    cargo install serial-monitor
    ```

    This will download the sources, compile and install the binary in
    `~/.cargo/bin` which you can add to your `PATH` if it is not already done.

    Now you can connect to the port with the following command:

    ```text
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

    People who do embedded development know it very well, you need to
    connect to the serial interface on the board. Don't look for some kind of
    slot, it's not a slot, it's just some of those pins.

    You can use the "USB to TTL Serial Cable". Connect to the pin on the board
    first and then plug the USB. The pins on your board should be labeled.
    Otherwise, check the documentation.

    I'm sure at this point you wonder how the hell do you connect a serial
    cable to the board. There are 4 colors: red (seems universally for power),
    black (seems universally for the ground), yellow/white (RXD), orange/green
    (TXD). The serial works by **NOT CONNECTING THE RED CABLE AT ALL**, then
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

    ```text
    serial-monitor -p /dev/ttyUSB1 -b 57600
    ```

    At this point you should see something on the terminal or press a key to
    make it react (the example code waits for a key).

Drawing pixels on the screen
----------------------------

### Ensure the screen is detected

Open the i2cdetect example and run it. The USB serial cable must be connected
before the board boots. You should see something like this on the terminal:

```text
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

```text
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

It is also possible that the cables of the serial moved a bit. Give it a few
tries.

### Ping the screen

If you take the example of blinking a led and mix it with the example of using
i2c, you can make a led that turns on when the screen is connected. In the API
of avr-hal you will find the function
[`ping_slave`](https://rahix.github.io/avr-hal/atmega32u4_hal/i2c/struct.I2c.html#method.ping_slave)
which returns a boolean if the slave answers to the ping. You will need the
address of the device which you can find somewhere in the documentation of the
screen. For the one we have chosen, this address is `0b0111100` (binary). All
the units of this screen share the same address, it is not unique.

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
tell the screen that this is a command and not data (pixels). This is
documented in the I2C protocol section of the documentation but since avr-hal
does already all the work for sending I2C commands, we only need to care about
the content of the packet.

> *After the transmission of the slave address, either the control byte or the
> data byte may be sent across the SDA. A control byte mainly consists of Co
> and D/C# bits following by six “0” ‘s.*
>
> *a. If the Co bit is set as logic “0”, the transmission of the following
> information will contain data bytes only.*
>
> *b. The D/C# bit determines the next data byte is acted as a command or a
> data. If the D/C# bit is set to logic “0”, it defines the following data
> byte as a command. If the D/C# bit is set to logic “1”, it defines the
> following data byte as a data which will be stored at the GDDRAM. The GDDRAM
> column address pointer will be increased by one automatically after each data
> write.*
>
> -- [The documentation](https://cdn.sparkfun.com/assets/1/a/5/d/4/DS-15890-Zio_OLED.pdf)

I did try to make sense of this but in the end I had to test empirically and
this is what worked:

 *  0b00000000 (8 zeroes): send a command
 *  0b01000000 (zero-one-6 zeroes): send data

A friend of mine later on told me that the Co bit actually means "more data is
to come" if set to `0`. This is his full explanation:

> *The "application" protocol of the OLED driver is byte oriented. The first
> byte you send, the control byte, tells the screen what it should expect for
> the remainder of the payload. The driver exposes two boolean toggles: Co and
> D/C#. These two toggles are laid out in the control byte as follows:*
>
> `Co Dx/C# p p p p p p`
>
> *`Co`: when set to 0 tells the screen that more data will follow after this
> byte and that it needs to keep reading. It is not explained in this snippet
> what happens if you set it to 1.*
>
> *`Dx/C#`: when set to 0 tells the screen that it needs to interpret the next
> byte as a command. When set to 1 it will store the following bytes in
> GDDRAM.*
>
> *`p`: padding. Our protocol is byte (8 bits) oriented and the two booleans only
> take up 2 bits total. We need to add 6 more bits to fill a full byte. These
> will always be 0.*

Let's try to turn on the screen by sending the command `0xaf`:

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

    // turn on the screen
    if let Err(err) = i2c.write(address, &[0b00000000, 0xaf]) {
        ufmt::uwriteln!(&mut serial, "Error: {:?}", err).void_unwrap();
    }

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

This should turn on the screen when the program starts. What you will see on
the screen is a random mess of pixels. This is what is in the memory of the
screen when you turn it on but since it is
[volatile](https://en.wikipedia.org/wiki/Volatile_memory), it can be really
anything.  Now we need to clear the screen.

### Filling up the screen

To fill up the screen we will first need to send a few commands to set the
coordinates of where we are going to draw. Then we send the pixels as data to
fill up the space.

If you check the list of commands in the manual you will find the command `15h`
(`0x15`) and `75h` (`0x75`) to set the column and the row respectively.

Both commands take two values: the start address and the end address. In other
words: `0x15` takes x1 and x2 while `0x75` takes y1 and y2 of the rectangle we
are going to draw.

Somewhere in the documentation you will also find that the different levels of
grey are actually coded on 4 bits. This actually means that 1 data byte is used
for 2 pixels on the screen. Therefore you will need to send half the bytes
than the number of pixels you are going to draw.

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

    // a small macro to help us send commands without repeating ourselves too much
    macro_rules! write_cmd {
        ($($bytes:expr),+) => {{
            if let Err(err) = i2c.write(address, &[0b00000000, $($bytes),+]) {
                ufmt::uwriteln!(&mut serial, "Error: {:?}", err).void_unwrap();
            }
        }};
    }

    // turn on the screen
    write_cmd!(0xaf);

    // fill the screen
    // our screen is 128 pixels long but we divide by 2 because there are 2 pixels per byte
    write_cmd!(0x15, 0, 63);
    // our screen is 128 pixels height
    write_cmd!(0x75, 0, 127);
    // we initialize an array of 64 + 1 bytes because 128 pixels / 2 + 1 byte for the control byte
    let mut data = [0xff; 65];
    data[0] = 0b01000000; // the control byte to send data
    for _ in 0..128 {
        if let Err(err) = i2c.write(address, &data) {
            ufmt::uwriteln!(&mut serial, "Error: {:?}", err).void_unwrap();
        }
    }
    // we should free the memory as it is quite limited
    drop(data);

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

This should fill your screen. You will see that this is not instant. The best
speed you can achieve is around 6 FPS. This is because this particular board
can not go faster than 400 kHz for its 2-wire serial interface (aka TWI) (this
is the I2C protocol).

It also seems to draw first the even lines and then the odd lines (or the other
way around). This can be changed using the command: `0xa0 0x51`.

This code could also be optimized by calling the minimum amount of time the
`write` method. The memory is limited so I personally used 2049 and called 4
times the `write` method to fill up the screen.

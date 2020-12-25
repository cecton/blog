---
title: "Rust and RISC-V"
date: 2020-12-15T06:21:00+02:00
---
**Target audience:** experienced developers with little knowledge in embedded
development and electronic. Check
["Rust, Arduino and Embedded Development as a Beginner: Part 1"]({{< ref "rust-and-arduino-part1" >}})
for a more beginner friendly article.

Introduction
------------

After my experience with the AVR microchip family (the same you use on Arduino
boards) I decided to try out Rust on a RISC-V based board. I don't work in
embedded development but most of the professional projects I saw were using
either AVR or ARM-based architectures. As I understand the reason for that
is because they are kinda industry standards and you can find PCB assemblers
who provide AVR and ARM chips more easily. RISC-V on the other hand is
relatively new (2010 according to
[Wikipedia](https://en.wikipedia.org/wiki/RISC-V)) and is entirely open source.
Unlike other instruction sets, RISC-V has no license fees attached to it.

I had the chance to test two different
chip producers for RISC-V: [SiFive](https://en.wikipedia.org/wiki/SiFive)
(which outsource the actual manufacturing) and
[GigaDevice](https://en.wikipedia.org/wiki/GigaDevice). SiFive is what you will
see on the most common development boards like the HiFive boards. In this
article I'm using the
[RED-V Thing Plus](https://www.sparkfun.com/products/15799) from SparkFun which
has the same MCU than the
[HiFive 1 Rev B](https://www.sifive.com/boards/hifive1-rev-b). One of the main
advantage of the RED-V is the
[Qwiic connector](https://www.sparkfun.com/qwiic) that I used in the previous
article ["Rust, Arduino and Embedded Development as a Beginner: Part 1"]({{< ref "rust-and-arduino-part1" >}}).
This allows us to connect the exact same screen
([Zio Qwiic OLED Display (1.5inch, 128x128)](https://www.sparkfun.com/products/15890))
to the board and get started very qwiickly. It is worth mentioning that while
the Qwiic connector makes it easy and safe to connect
components together, it is not fast: connection through
serial is faster. In this tutorial, we are going to push this I2C to its
limits thanks to the very powerful FE310 microchip.

### My own hardware choice

 *  [SparkFun RED-V Thing Plus](https://www.sparkfun.com/products/15799)
 *  [Zio Qwiic OLED Display (1.5inch, 128x128)](https://www.sparkfun.com/products/15890)
 *  [Qwiic Cable - 50mm](https://www.sparkfun.com/products/14426)

**Note:** We don't need a USB to TTL serial cable this time.

First step: making sure everything works
----------------------------------------

I assume that the screen is working because I already made it work in the
previous tutorial with the AVR. All we need to do now is make sure that the
board itself is working by making a led blink. In this tutorial I'm not going
to go through installing the IDE as suggested by the
[SparkFun's RED-V development guide](https://learn.sparkfun.com/tutorials/red-v-development-guide)
because I feel more confident with my previous experience and I go directly to
the
[Rust guide to get started with HiFive 1 boards](https://github.com/riscv-rust/riscv-rust-quickstart).
It is very beginner friendly and, as it turns out, incredibly easy to get
started. This is because the bootloader and the flashing program are using
JTag.

There are two ways to flash the firmware:

1.  Using J-Link GDB Server

    You will need to download and install the
    [J-Link GDB Server](https://www.segger.com/downloads/jlink/). I'm not
    really sure how to install it as I used the ArchLinux AUR package
    [jlink-software-and-documentation](https://aur.archlinux.org/packages/jlink-software-and-documentation/)
    (thanks to Alexis Polti for the maintenance). This method doesn't require
    you to press a button to flash but it does require to install the software.

2.  By copying the hex file to the disk.

    You can actually boot in flash mode by pressing the button. A "USB disk"
    will appear on your system: mount it, copy your hex file, unmount, reboot.
    No software needed.

This time I followed the tutorial and I created a new repository based on the
template. Then I edited the `leds_blink.rs` example to adapt it to my board.
The RED-V Thing Plus doesn't have three leds like the HiFive 1, therefore I
adapted the example to a single led located on the SPI SCK pin. I found this
led by looking at the
[schematics](https://cdn.sparkfun.com/assets/a/c/3/e/4/RedVThingPlus.pdf)
that you can find in the documents on SparkFun. I looked for "led" and I found
a blue led between IO23 and SPI MOSI. You can always check if the code builds
by using `cargo check`. In this case, use `cargo check --example leds_blink`
to check that the example builds.

```rust
#![no_std]
#![no_main]

/*
* Basic blinking LEDs example using mtime/mtimecmp registers
* for "sleep" in a loop. Blinks each led once and goes to the next one.
*/

extern crate panic_halt;

use hifive1::hal::delay::Sleep;
use hifive1::hal::prelude::*;
use hifive1::hal::DeviceResources;
use hifive1::sprintln;
use hifive1::{pin, pins, Led};
use riscv_rt::entry;

#[entry]
fn main() -> ! {
    let dr = DeviceResources::take().unwrap();
    let p = dr.peripherals;
    let pins = dr.pins;

    // Configure clocks
    let clocks = hifive1::clock::configure(p.PRCI, p.AONCLK, 320.mhz().into());

    // Configure UART for stdout
    hifive1::stdout::configure(
        p.UART0,
        pin!(pins, uart0_tx),
        pin!(pins, uart0_rx),
        115_200.bps(),
        clocks,
    );

    // get all 3 led pins in a tuple (each pin is it's own type here)
    let rgb_pins = pins!(pins, (spi0_sck));
    let mut blue = rgb_pins.into_inverted_output();

    // get the local interrupts struct
    let clint = dr.core_peripherals.clint;

    let mut led_status = true;

    // get the sleep struct
    let mut sleep = Sleep::new(clint.mtimecmp, clocks);

    sprintln!("Starting blink loop");

    loop {
        match led_status {
            true => blue.set_high().unwrap(),
            false => blue.set_low().unwrap(),
        }

        led_status = !led_status;

        sprintln!("Status: {}", led_status);

        sleep.delay_ms(200_u32);
    }
}
```

If the check is working you can now use one of those methods to upload the new
firmware:

 *  **Method 1:** Using J-Link GDB Server

    You will need to run multiple terminal for this because the GDB server
    needs to keep running in the background. In one terminal you need to run:

    ```text
    sudo JLinkGDBServer -device FE310 -if JTAG -speed 4000 -port 3333 -nogui
    ```

    (You might not need `sudo` if you have set up user permissions to the
    device).

    The program should keep waiting if it detected the device properly.

    Then you can run in another terminal:

    ```text
    cargo run --example leds_blink
    ```

    The two programs will interact and load the new firmware. You should see
    the led blinking.

 *  **Method 2:** By copying the hex file.

    1.  First, you need to build the program:

        ```text
        cargo build --example leds_blink
        ```

    2.  Now you can connect the device in USB and you should see a new disk,
        mount it.

    3.  Then generate the hex file from the binary using this command:

        ```text
        objcopy -S -j .text -j .rodata -O ihex \
            ./target/riscv32imac-unknown-none-elf/debug/examples/leds_blink \
            /media/your_device/leds_blink.hex
        ```

    4.  Unmount. It should reboot automatically on the new firmware and you
        should see the led blinking.

You can also observe the logs through a serial USB. Check `/dev/ttyACM0`.
Example using [serial-monitor](https://crates.io/crates/serial-monitor):

```text
serial-monitor -p /dev/ttyACM0 -b 115200
```

Make an animation on the screen
-------------------------------

If you followed the previous tutorial
["Rust, Arduino and Embedded Development as a Beginner: Part 1"]({{< ref "rust-and-arduino-part1" >}})
this should be quite straightforward.

You will need the animation frame we have generated in
["Rust, Arduino and Embedded Development as a Beginner: Part 3"]({{< ref "rust-and-arduino-part3" >}}).
Copy the RAW files to the `src/` directory.

For the example, we are using nightly features, so make sure you tell rustup to use it:

```text
rustup override set nightly
```

Now in the file `src/main.rs`:

```rust
#![no_std]
#![no_main]

extern crate panic_halt;

use hifive1::hal::i2c::{I2c, Speed};
use hifive1::hal::prelude::*;
use hifive1::hal::DeviceResources;
use hifive1::{pin, sprintln};
use riscv_rt::entry;

const FRAME_1: &[u8] = include_bytes!("F501-1.raw");
const FRAME_2: &[u8] = include_bytes!("F501-2.raw");
const FRAME_3: &[u8] = include_bytes!("F501-3.raw");
const FRAME_4: &[u8] = include_bytes!("F501-4.raw");
const FRAME_5: &[u8] = include_bytes!("F501-5.raw");
const FRAME_6: &[u8] = include_bytes!("F501-6.raw");
const FRAME_7: &[u8] = include_bytes!("F501-7.raw");
const FRAME_8: &[u8] = include_bytes!("F501-8.raw");
const FRAME_9: &[u8] = include_bytes!("F501-9.raw");
const FRAME_10: &[u8] = include_bytes!("F501-10.raw");
const FRAME_11: &[u8] = include_bytes!("F501-11.raw");
const FRAME_12: &[u8] = include_bytes!("F501-12.raw");
const FRAME_13: &[u8] = include_bytes!("F501-13.raw");
const FRAME_14: &[u8] = include_bytes!("F501-14.raw");
const FRAME_15: &[u8] = include_bytes!("F501-15.raw");

#[entry]
fn main() -> ! {
    let dr = DeviceResources::take().unwrap();
    let p = dr.peripherals;
    let pins = dr.pins;

    // Configure clocks
    // NOTE: this screen https://www.sparkfun.com/products/15890 goes up to 320MHz
    // NOTE: watch out because it gets really warm!
    let clocks = hifive1::clock::configure(p.PRCI, p.AONCLK, 100.mhz().into());

    // Configure UART for stdout
    hifive1::stdout::configure(
        p.UART0,
        pin!(pins, uart0_tx),
        pin!(pins, uart0_rx),
        115_200.bps(),
        clocks,
    );

    // Configure I2C
    let sda = pin!(pins, i2c0_sda).into_iof0();
    let scl = pin!(pins, i2c0_scl).into_iof0();
    let mut i2c = I2c::new(p.I2C0, sda, scl, Speed::Fast, clocks);

    // Get blue led
    let sck = pin!(pins, spi0_sck);
    let mut blue = sck.into_inverted_output();

    // This is the the address of your device. You can find it in the data sheets of
    // your device.
    let address = 0b0111100;

    // a small macro to help us send commands without repeating ourselves too much
    macro_rules! write_cmd {
        ($($bytes:expr),+) => {{
            if let Err(err) = i2c.write(address, &[0b00000000, $($bytes),+]) {
                sprintln!("Error: {:?}", err);
            }
        }};
    }

    // turn on the screen
    write_cmd!(0xae);
    write_cmd!(0xaf);
    write_cmd!(0xa0, 0x51);

    // fill the screen
    // our screen is 128 pixels long but we divide by 2 because there are 2 pixels per byte
    write_cmd!(0x15, 0, 63);
    // our screen is 128 pixels height
    write_cmd!(0x75, 0, 127);
    // note that I reduced the buffer size!!
    let mut data = [0x00; 1024 + 1];
    data[0] = 0b01000000; // the control byte
    for _ in 0..8 {
        if let Err(err) = i2c.write(address, &data) {
            sprintln!("Error: {:?}", err);
        }
    }

    // dimensions of the frames
    let width = 40;
    let height = 42;

    // prepare drawing area
    write_cmd!(0x15, 0, width / 2 - 1);
    write_cmd!(0x75, 0, height - 1);

    // we override the first data byte with the control byte which tells the screen we are
    // sending data
    //
    // note: it was done already before but just want to make sure in case you comment the screen
    // filling above
    data[0] = 0b01000000;

    // a helper to help us draw an image
    let mut draw_frame = |frame: &[u8]| {
        // an iterator that will convert the frame's bytes to data bytes usable by the screen:
        //
        // every byte sent to the screen draws 2 pixels: the first 4 bits are for the first
        // pixel while the last 4 bits are for the second pixel
        //
        // every byte in the frame contains 8 bits so 8 monochromatic pixels
        //
        // 8 / 2 = 4
        //
        // this iterator returns 4 data bytes for 1 frame byte
        let mut chunks = frame.iter().map(|x| {
            [
                (x & 0b10000000).count_ones() as u8 * 0b11110000
                    + (x & 0b01000000).count_ones() as u8 * 0b00001111,
                (x & 0b00100000).count_ones() as u8 * 0b11110000
                    + (x & 0b00010000).count_ones() as u8 * 0b00001111,
                (x & 0b00001000).count_ones() as u8 * 0b11110000
                    + (x & 0b00000100).count_ones() as u8 * 0b00001111,
                (x & 0b00000010).count_ones() as u8 * 0b11110000
                    + (x & 0b00000001).count_ones() as u8 * 0b00001111,
            ]
        });
        // we count the number of bytes that have been copied so we don't send the whole buffer
        let mut i = 1;
        while let Some(chunk) = chunks.next() {
            // copy_from_slice requires that the source slice and the destination slice are
            // exactly the same otherwise it will panic
            data[i..(i + 4)].copy_from_slice(&chunk);
            i += 4;
        }

        if let Err(err) = i2c.write(address, &data[..i]) {
            sprintln!("Error: {:?}", err);
        }
        let _ = blue.toggle();
    };

    loop {
        draw_frame(FRAME_1);
        draw_frame(FRAME_2);
        draw_frame(FRAME_3);
        draw_frame(FRAME_4);
        draw_frame(FRAME_5);
        draw_frame(FRAME_6);
        draw_frame(FRAME_7);
        draw_frame(FRAME_8);
        draw_frame(FRAME_9);
        draw_frame(FRAME_10);
        draw_frame(FRAME_11);
        draw_frame(FRAME_12);
        draw_frame(FRAME_13);
        draw_frame(FRAME_14);
        draw_frame(FRAME_15);
    }
}
```

Implementation notes
--------------------

In
["Rust, Arduino and Embedded Development as a Beginner: Part 3"]({{< ref "rust-and-arduino-part3" >}})
we used a macro to draw every frame. We could have done it using a function or
a closure but that would mean that when the code is compiled to assembly, there
will be jumps to this function/closure. I wanted to optimize the best I could
so I used a macro instead.

A macro literally add code to your code before it gets compiled. So for
example, if you do:

```rust
macro_rules hello_world {
    () => {{
        println!("Hello World!");
    }};
}

hello_world!();
hello_world!();
hello_world!();
```

This code will be equivalent to:

```rust
println!("Hello World!");
println!("Hello World!");
println!("Hello World!");
```

If, instead, you use a function, the code will look like this:

```rust
fn hello_world() {
    println!("Hello World!"); // this code exists only once
}

hello_world(); // jump to hello_world
hello_world(); // jump to hello_world
hello_world(); // jump to hello_world
```

You can see the generated code of a macro by using
[cargo-expand](https://crates.io/crates/cargo-expand). This is a very handy
tool if you work with macros and want to see what's going on.

You can read more about the cost of a jump
[here](https://stackoverflow.com/questions/5127833/meaningful-cost-of-the-jump-instruction).
On a normal CPU it would be ridiculous to do but on the ATmega I thought (I
might be wrong) that it could have a bigger impact. My intent was to make it as
fast as possible.

In this example with RISC-V I decided to not use a macro this time but a
closure instead. My intent is still to make it as fast as possible but there is
a little detail you need to know about the FE310-G002: this MCU has an
instruction cache which can overflow if you have too many instructions in your
loop, thus making cache miss and forcing the MCU to load the instructions from
flash memory instead (which is slower than the cache). In this particular case
the macro version of `draw_frame` was a lot slower than the closure.

We didn't have this issue with the ATmega because there is no MCU cache, it
just loads everything from flash. On modern CPUs the entire program is loaded
into memory and there is a much bigger CPU cache.

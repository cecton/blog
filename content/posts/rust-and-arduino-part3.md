---
title: "Rust, Arduino and Embedded Development as a Beginner: Part 3"
date: 2020-09-04T21:07:00+02:00
---
Introduction
------------

After reading the [part 2]({{< ref "rust-and-arduino-part2" >}}), our
adventurers finally entered in the last step of the journey (or at least
that's what they think) and are looking forward to...

Make an animation on the screen
-------------------------------

It is possible to make a small animation on this screen simply by drawing
images one by one. For that we will need to download a small monochromic
animation. In this example I'm using
[this small falling star animation from itch.io](https://kvsr.itch.io/falling-star).

### Make a raw format

Since we only need black or white pixels we could encode every pixel as a bit.
With this technique we can encode 8 pixels per byte. The tool
[ImageMagick](https://imagemagick.org/index.php) can help us convert those
images:

```bash
for i in {1..15}
do
    convert examples/F501-$i.png -background black -filter Box \
        -define filter:blur=0 -resize 33x42 -monochrome -depth 1 \
        gray:examples/F501-$i.raw
done
```

Note that for some reason there is a padding on every row: every row of the
image must have a whole number of bytes. Therefore, our image of 33 pixels per
row is converted to 40 pixels (40x42) which makes 210 bytes in total
(`40 * 42 / 8`).

### Draw images one by one

Now that we have `*.raw` files, we can include them in the program by using
`const`. After adding a bit of code to convert our raw format to data bytes
that the screen can read, you should see an animation on your screen.

```rust
#![no_std]
#![no_main]

extern crate panic_halt;
use arduino_leonardo::prelude::*;

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
    write_cmd!(0xa0, 0x51);

    // fill the screen
    // our screen is 128 pixels long but we divide by 2 because there are 2 pixels per byte
    write_cmd!(0x15, 0, 63);
    // our screen is 128 pixels height
    write_cmd!(0x75, 0, 127);
    let mut data = [0x00; 1024 + 1];
    data[0] = 0b01000000; // the control byte
    for _ in 0..8 {
        if let Err(err) = i2c.write(address, &data) {
            ufmt::uwriteln!(&mut serial, "Error: {:?}", err).void_unwrap();
        }
    }

    // dimension of the frames
    let width = 40;
    let height = 42;

    // prepare drawing area
    write_cmd!(0x15, 0, width / 2 - 1); // 2 pixels per data byte
    write_cmd!(0x75, 0, height - 1);

    // we override the first data byte with the control byte which tells the screen we are
    // sending data
    //
    // note: it was already done before but I just want to make sure in case you comment the
    // screen filling above
    data[0] = 0b01000000;

    // a not-so-small macro to help us draw an image
    macro_rules! draw_frame {
        ($frame:expr) => {{
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
            let mut chunks = $frame.iter().map(|x| {
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
                // exactly of the same size otherwise it will panic
                data[i..(i+4)].copy_from_slice(&chunk);
                i += 4;
            }

            if let Err(err) = i2c.write(address, &data[..i]) {
                ufmt::uwriteln!(&mut serial, "Error: {:?}", err).void_unwrap();
            }
            delay.delay_ms(1000u16); // some delay so we have the time to see the frame
            led_rx.toggle().void_unwrap();
        }};
    }

    loop {
        draw_frame!(FRAME_1);
        draw_frame!(FRAME_2);
        draw_frame!(FRAME_3);
        draw_frame!(FRAME_4);
        draw_frame!(FRAME_5);
        draw_frame!(FRAME_6);
        draw_frame!(FRAME_7);
        draw_frame!(FRAME_8);
        draw_frame!(FRAME_9);
        draw_frame!(FRAME_10);
        draw_frame!(FRAME_11);
        draw_frame!(FRAME_12);
        draw_frame!(FRAME_13);
        draw_frame!(FRAME_14);
        draw_frame!(FRAME_15);
    }
}
```

### Troubleshooting

1.  *I have this error.*

    ```
    error: couldn't read boards/arduino-leonardo/examples/F501-15.raw: No such file or directory (os error 2)
      --> boards/arduino-leonardo/examples/leonardo-i2cdetect.rs:21:25
       |
    21 | const FRAME_15: &[u8] = include_bytes!("F501-15.raw");
       |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
       |
       = note: this error originates in a macro (in Nightly builds, run with -Z macro-backtrace for more info)
    ```

    That's because the frame files are probably missing. If you used the
    example directly, those file should be located near the `i2cdetect.rs` file
    in the example directory. The macro `include_bytes!` looks for the file
    relative to where the source file is
    ([see doc](https://doc.rust-lang.org/std/macro.include_bytes.html)).

2.  *I think the first 6 frames show up correctly but the 7th is glitched and
    all the rest is glitched.*

    No panic, this is expected for this example.

    The reason why it is glitched is because we have a "stack corruption". In
    other words: the
    [stack memory](https://doc.rust-lang.org/stable/rust-by-example/std/box.html)
    got corrupted. In this case it happens because
    [our MCU](http://cdn.sparkfun.com/datasheets/Dev/Arduino/Boards/ATMega32U4.pdf)
    (microcontroller unit / board) has only 2500 bytes of memory and we are
    storing 15 images of 210 bytes (3150 bytes) in it.

    I personally thought that
    [using `const` instead of `static`](https://doc.rust-lang.org/1.20.0/book/first-edition/const-and-static.html#static)
    would help with the allocations because `consts` are actually inlined so I
    would have expected that the allocation would be only at the place it is
    used and it would be freed when leaving the scope of the block but this is
    not how it works.

    On an x86 architecture the programs are actually loaded entirely into
    memory before being executed. Because the consts are part of the program,
    they are normally loaded into memory with the rest of the code. We say that
    x86 (and ARM) has "one address space". It means that the assembly code provides
    instructions to access only one address space. This address space is the
    memory (RAM). People sometimes call the single address space model
    "[Von Neumann architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture)"
    or "Princeton architecture" but this not 100% accurate, as they really
    refer to whether there are separate memories, not whether they have a
    single address space.

    Here on the [AVR](https://en.wikipedia.org/wiki/AVR_microcontrollers)
    the program is actually stored and executed from a flash memory (16kB on
    [our MCU](http://cdn.sparkfun.com/datasheets/Dev/Arduino/Boards/ATMega32U4.pdf))
    and therefore is not loaded in memory
    ([SRAM](https://en.wikipedia.org/wiki/Static_random-access_memory)). We say
    that AVR has "two address spaces". For example the address 1234 in flash
    memory also exists in SRAM. People sometimes call the separate model
    "[Harvard architecture](https://en.wikipedia.org/wiki/Harvard_architecture)".
    The SRAM is normally used for stack and heap memory during the execution.

    Ideally we want to only load what we need to our SRAM, for instance one
    frame at a time, the rest should stay in flash memory. AVR actually has
    instructions to load bytes from the flash memory to the SRAM (it even has
    instructions for saving to flash). Unfortunately the Rust compiler has been
    designed in a way that the consts are actually loaded into memory when the
    program starts.

    It is important to note that on ARM microcontrollers for example both flash
    and RAM are in the same address space. For example, they could have decided
    that addresses 0 through 1023 are RAM, and 1024 to 2047 are flash.

3.  *How do we fix?*

    We don't. It's probably very interesting to go deeper, learn more about
    assembly and the Rust compiler but this is out of scope. I might come back
    to it at some point but I also bought a
    [RISC-V board](https://www.sparkfun.com/products/15799)
    which is much more powerful (16kB data SRAM!).
    [RISC-V](https://en.wikipedia.org/wiki/RISC-V)
    is also particularly interesting because it is an entirely open source instruction set,  so we
    could explore it even deeper.

Conclusion
----------

You can make animations with the ATmega32u4 but you'll need to do some
assembly.

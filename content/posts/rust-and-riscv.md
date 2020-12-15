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
either AVR either ARM-based architecture. As I understand the reason for that
is because they are kinda industry standards and you can find PCB assemblers
who provides AVR and ARM chips more easily. RISC-V on the other hand is
relatively new (2010 according to
[Wikipedia](https://en.wikipedia.org/wiki/RISC-V)) but is entirely open source
which doesn't require fees to use it.

I had the chance to test two different
chip producers for RISC-V: [SiFive](https://en.wikipedia.org/wiki/SiFive)
(which actually outsource the actual manufacturing) and
[GigaDevice](https://en.wikipedia.org/wiki/GigaDevice). SiFive is what you will
see on the most common development boards like the HiFive boards. In this
article I'm using the
[RED-V Thing Plus](https://www.sparkfun.com/products/15799) from SparkFun which
has the same MCU than the
[HiFive 1 Rev B](https://www.sifive.com/boards/hifive1-rev-b). One of the main
advantage of the RED-V is again the
[Qwiic connector](https://www.sparkfun.com/qwiic) that I use in the previous
article ["Rust, Arduino and Embedded Development as a Beginner: Part 1"]({{< ref "rust-and-arduino-part1" >}}).
This allows us to connect the exact same screen
([Zio Qwiic OLED Display (1.5inch, 128x128)](https://www.sparkfun.com/products/15890))
to the board and get started very qwiickly. It is worth mentioning that the
advantage of the Qwiic connector is how easily and safely you can connect
components together but not the transfer speed of the data: connection through
serial is faster. In this tutorial, we are going to push this I2C to its
limits thanks to the very powerful FE310 microchip.

### My own hardware choice

 *  [SparkFun RED-V Thing Plus](https://www.sparkfun.com/products/15799)
 *  [Zio Qwiic OLED Display (1.5inch, 128x128)](https://www.sparkfun.com/products/15890)
 *  [Qwiic Cable - 50mm](https://www.sparkfun.com/products/14426)

**Note:** We don't need a USB to TTL serial cable this time.

First step: making sure everything works
----------------------------------------

I assume that the screen is working because I already made it working in the
previous tutorial with the AVR. All we need to do now is making sure that the
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

If the check is working you can now use one of those methods to upload the new
firmware:

1.  Using J-Link GDB Server

    You will need to run multiple terminal for this because the GDB server
    needs to keep running in the background. In one terminal you need to run:

    ```
    sudo JLinkGDBServer -device FE310 -if JTAG -speed 4000 -port 3333 -nogui
    ```

    (You might not need `sudo` if you have set up user permissions to the
    device).

    The program should keeps waiting if it detected the device properly.

    Then you can run in another terminal:

    ```
    cargo run --example leds_blink
    ```

    The two programs will interact and load the new firmware. You should see
    the led blinking.

2.  By copying the hex file.

    You first need to build the program:

    ```
    cargo build --example leds_blink
    ```

    Then press the button on the board to enter flash mode. A new "USB disk"
    will appear. Mount it.

    Now you can generate the hex file from the binary using this command:

    ```
    objcopy -S -j .text -j .rodata -O ihex \
        ./target/riscv32imac-unknown-none-elf/debug/examples/leds_blink \
        /media/your_device/leds_blink.hex
    ```

    Unmount. It should reboot automatically on the new firmware and you should
    see the led blinking.

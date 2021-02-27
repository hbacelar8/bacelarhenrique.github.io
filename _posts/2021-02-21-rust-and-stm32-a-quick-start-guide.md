---
title:  "Rust and STM32: A Quick Start Guide"
excerpt: A quick start guide to get you working on STM32 development with embedded Rust.
tags: stm32 rust-embedded programming
last_modified_at: "2021-02-27"
header:
  teaser: assets/images/rust.jpg
  overlay_image: assets/images/rust.jpg
  overlay_filter: 0.5
---

# Preface
This article aims to explain the basics to get you developing on STM32 devices using Rust language. This tutorial takes place in a linux machine, precisely Ubuntu.

# Why Rust?
Rust claims to be an interesting alternative for embedded developing for having, among others, characteristics such as:

- **Powerful static analysis**: Enforce pin and peripheral configuration at compile time, guaranteeing that resources won’t be used by unintended parts of your application.
- **Flexible memory**: Dynamic memory allocation is optional. Use a global allocator and dynamic data structures. Or leave out the heap altogether and statically allocate everything.
- **Fearless concurrency**: Rust makes it impossible to accidentally share state between threads. Use any concurrency approach you like, and you’ll still get Rust’s strong guarantees.

# What will you need?
- A PC with the Rust Toolchain installed
- A board with a STM32 device

In this case, I'll be working with the NUCLEO-F103RB development board containing a STM32F103RB MCU. This board already comes with an onboard ST-LINK debugger and programmer, but you can use an external one if needed.

# Installations
As said before, all the steps described is this tutorial are for a machine running linux (Ubuntu).

## Rust

Start by making sure you have ```curl``` installed

```bash
sudo apt install curl
```

Then run the following to install Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

After installation, you can check your Rust compiler version with

```bash
rustc -V
```

Rust installation already comes with Cargo, the Rust's *package manager* that will be used to download Rust package's dependencies, compile the packages, make them distributable and upload them to [crates.io](https://crates.io/){:target="_blank"}.

For bandwidth and disk usage concerns, the default installation only supports native compilation. To add cross compilation support for the Arm Cortex-M architectures, you'll have to install them separately.

For Cortex-M0, M0+, and M1 (ARMv6-M architecture):

``` bash
rustup target add thumbv6m-none-eabi
```

For Cortex-M3 (ARMv7-M architecture):

```bash
rustup target add thumbv7m-none-eabi
```

For Cortex-M4 and M7 without hardware floating point (ARMv7E-M architecture):

```bash
rustup target add thumbv7em-none-eabi
```

And finally, for Cortex-M4F and M7F with hardware floating point (ARMv7E-M architecture):

```bash
rustup target add thumbv7em-none-eabihf
```

## Cargo binary utils
Make sure you have the ```build-essential``` package installed, which is necessary for having the C compiler used by Cargo

```bash
sudo apt install build-essential
```

Then you can install ```cargo-binutils```

```bash
cargo install cargo-binutils
```

```bash
rustup component add llvm-tools-preview
```

## Cargo flash
Cargo flash will be used to program the microcontroller. Before installing it, you must first install the ```libusb``` package

```bash
sudo apt install libusb-1.0-0-dev
```

And finally, for ```cargo flash```

```bash
cargo install cargo-flash
```

# Creating the project
Let's create a simple blink project. By running the following command, a new Rust project will be created in a new folder

```bash
cargo init blink && cd blink
```

## Configurations
Before writing any code, you must first configure the project so that it compiles for your specific target and uses the correct linker script. Since I'm working with a Cortex M3, I must configure it for the ```thumbv7m-none-eabi``` target. This can be done by creating a file ```.cargo/config``` in your project root.

```conf
# .cargo/config

[build]
# Always compile for the instruction set of the STM32F1
target = "thumbv7m-none-eabi"

# use the Tlink.x scrip from the cortex-m-rt crate
rustflags = [ "-C", "link-arg=-Tlink.x"]
```

The next thing we need to do is create the linker script responsible for telling the linker about the memory layout of the device. It must be created in the project root and named ```memory.x```. The following configuration is for the STM32F103RB MCU which has 128 Kbytes of Flash memory and 20 Kbytes of SRAM. You must adapt it to your device if it's the case.

```
/* memory.x - Linker script for the STM32F103RB */

MEMORY
{
    /* Flash memory begins at 0x80000000 and has a size of 128kB*/
    FLASH : ORIGIN = 0x08000000, LENGTH = 128K
    /* RAM begins at 0x20000000 and has a size of 20kB*/
    RAM : ORIGIN = 0x20000000, LENGTH = 20K
}
```

Now, you need to configure the ```Cargo.toml``` file already present in your project root. This is the file where all the dependencies (crates) used in your project must be specified.

For this project in particular, copy the following content to the file

```
# Cargo.toml
[package]
name = "blink"
version = "0.1.0"
edition = "2018"

[profile.release]
opt-level = 'z' # turn on maximum optimizations. We only have 64kB
lto = true      # Link-time-optimizations for further size reduction

[dependencies]
cortex-m = "^0.6.3"       # Access to the generic ARM peripherals
cortex-m-rt = "^0.6.12"   # Startup code for the ARM Core
embedded-hal = "^0.2.4"   # Access to generic embedded functions (`set_high`)
panic-halt = "^0.2.0"     # Panic handler

# Access to the STM32F103 HAL.
[dependencies.stm32f1xx-hal]
# STM32F103RB contains a 128kB flash variant which is called "medium density"
features = ["stm32f103", "rt", "medium"]
version = "^0.6.1"
```

## Main project
And finally, for the ```main.rs``` file, a simple blink program

```rust
// src/main.rs

// std and main are not available for bare metal software
#![no_std]
#![no_main]

use panic_halt as _;

use cortex_m_rt::entry;
use embedded_hal::digital::v2::OutputPin;
use stm32f1xx_hal as hal;
use hal::{pac, delay::Delay, prelude::*};

#[entry]
fn main() -> ! {
    /* Get access to device and core peripherals */
    let dp = pac::Peripherals::take().unwrap();
    let cp = cortex_m::Peripherals::take().unwrap();

    /* Get access to RCC, AFIO and GPIOA */
    let mut rcc = dp.RCC.constrain();
    let mut flash = dp.FLASH.constrain();
    let mut gpioa = dp.GPIOA.split(&mut rcc.apb2);

    /* Set up LED pin */
    let mut led = gpioa.pa5.into_push_pull_output(&mut gpioa.crl);

    /* Set up sysclk and freeze it */
    let clocks = rcc.cfgr.sysclk(8.mhz()).freeze(&mut flash.acr);

    /* Set up systick delay */
    let mut delay = Delay::new(cp.SYST, clocks);

    loop {
        /* Light show */
        led.set_high().ok();
        delay.delay_ms(1_000_u16);
        led.set_low().ok();
        delay.delay_ms(1_000_u16);
    }
}
```

# Building and Flashing
To build the project in release mode

```bash
cargo build --release
```

Now, you are ready to flash the program. With the device connected and the ST-LINK recognized, flash it by specifying the device with

```bash
cargo flash --chip stm32f103rb --release
```

# Go further
Some links used as reference that will definitely be useful on your projects:

- [The ```stm32f1xx_hal``` crate documentation](https://docs.rs/stm32f1xx-hal/0.7.0/stm32f1xx_hal/index.html){:target="_blank"}
- [Some examples using the ```stm32f1xx_hal``` crate](https://github.com/stm32-rs/stm32f1xx-hal/tree/master/examples){:target="_blank"}
- [My own repository with some Rust examples using RTIC](https://github.com/bacelarhenrique/rust-embedded-examples){:target="_blank"}
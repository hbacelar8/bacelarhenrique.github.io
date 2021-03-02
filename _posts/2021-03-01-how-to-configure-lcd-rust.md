---
title:  "How To Configure LCD in 4-Bit Mode on STM32 With Rust"
excerpt: How to configure an 16x2 LCD display in 4-bit mode on a STM32 MCU using Rust language.
tags: rust-embedded stm32 programming
last_modified_at: "2021-03-01"
header:
  teaser: assets/images/lcd_display.jpg
  overlay_image: assets/images/lcd_display.jpg
  overlay_filter: 0.5
---

# Preface
LCD modules are very commonly used in most embedded projects, the reason being its cheap price, availability and programmer friendly. Most of us would have come across these displays in our day to day life, either at PCO’s or calculators. 16×2 LCD is named so because it has 16 columns and 2 rows. There are a lot of combinations available like: 8×1, 8×2, 10×2, 16×1, etc. However, the most used one is the 16×2 LCD.

Below I explain how to interface this module in 4-bit mode, meaning only four pins from the microcontroller we'll be required to send data or commands to the LCD. I'll be using a STM32F103RB MCU and the code we'll be written in Rust, using the ```stm32f1xx-hal``` crate.

# Code
As mentioned before, I'll be using the ```stm32f1xx-hal``` crate, whose documentation can be found [here](https://docs.rs/stm32f1xx-hal/0.7.0/stm32f1xx_hal/){:target="_blank"}.

## Imports
In a new Rust file, namely ```lcd.rs```, we start by importing the crates we'll be using.

```rust
// src/lcd.rs
// LCD module

use cortex_m::asm::nop;
use embedded_hal::digital::v2::OutputPin;
use panic_halt as _;

use hal::{
    delay::Delay,
    gpio::{
        gpioa::{PA0, PA1},
        gpioc::{PC0, PC1, PC2, PC3},
        Output, PushPull,
    },
    prelude::*,
};
use stm32f1xx_hal as hal;
```

## LCD ```struct```
Next, we create a public ```struct``` with all the pins that'll be used and a delay instance.

```rust
pub struct LCD {
    rs: PA0<Output<PushPull>>,
    en: PA1<Output<PushPull>>,
    d4: PC0<Output<PushPull>>,
    d5: PC1<Output<PushPull>>,
    d6: PC2<Output<PushPull>>,
    d7: PC3<Output<PushPull>>,
    delay: Delay,
}
```

The implementation of this structure is as follows

```rust
impl LCD {
    pub fn new(
        rs: PA0<Output<PushPull>>,
        en: PA1<Output<PushPull>>,
        d4: PC0<Output<PushPull>>,
        d5: PC1<Output<PushPull>>,
        d6: PC2<Output<PushPull>>,
        d7: PC3<Output<PushPull>>,
        delay: Delay,
    ) -> LCD {
        LCD {
            rs,
            en,
            d4,
            d5,
            d6,
            d7,
            delay,
        }
    }

    /* -------------------- Private Functions -------------------- */

    /* ... */

    /* -------------------- Public Functions -------------------- */

    /* ... */
}
```

When calling the ```new()``` function, a new instance of the LCD ```struct``` is created.

## Private functions
Let's add two private functions which will be frequently used by other functions. The first one, namely ```_pulse_enable()```, generates a pulse on the enable pin.

```rust
/// Pulse enable
fn _pulse_enable(&mut self) {
    self.en.set_high().ok();
    nop();
    self.en.set_low().ok();
}
```

The second one, namely ```_send_nibble()```, sends half a byte to the LCD through the data pins.

```rust
/// Send half a byte to the LCD
///
/// # Arguments
///
/// * `nibble` - Nibble to send
fn _send_nibble(&mut self, nibble: u8) {
    if ((nibble >> 3) & 0x01) == 0x01 {
        self.d7.set_high().ok();
    } else {
        self.d7.set_low().ok();
    }

    if ((nibble >> 2) & 0x01) == 0x01 {
        self.d6.set_high().ok();
    } else {
        self.d6.set_low().ok();
    }

    if ((nibble >> 1) & 0x01) == 0x01 {
        self.d5.set_high().ok();
    } else {
        self.d5.set_low().ok();
    }

    if ((nibble >> 0) & 0x01) == 0x01 {
        self.d4.set_high().ok();
    } else {
        self.d4.set_low().ok();
    }

    self._pulse_enable();
}
```

## Public functions

Now, we can create a public initialization function which will do all the configurations required to initialize the display, including configuring it in 4-bit mode.

```rust
/// Initialize the LCD
pub fn init(&mut self) {
    // Power on delay
    self.delay.delay_ms(100_u16);

    // Send command
    self.rs.set_low().ok();

    // First nibble 0b0011
    self._send_nibble(0x03);
    self.delay.delay_us(4100_u16);

    // Second nibble 0b0011
    self._pulse_enable();
    self.delay.delay_us(100_u16);

    // Third nibble 0b0011
    self._pulse_enable();
    self.delay.delay_us(100_u16);

    // Configure LCD in 4-bit mode
    self._send_nibble(0x02);
    self.delay.delay_us(100_u16);

    // Function set to configure the interface, number of lines and the font
    self.send_cmd(0x28);
    self.delay.delay_us(53_u16);

    // Display off
    self.send_cmd(0x08);
    self.delay.delay_us(53_u16);

    // Clear display (demands a longer delay)
    self.send_cmd(0x01);
    self.delay.delay_us(3000_u16);

    // Entry mode set
    self.send_cmd(0x06);
    self.delay.delay_us(53_u16);

    // Display on
    self.send_cmd(0x0C);
    self.delay.delay_us(53_u16);
}
```

Finally, we can add other general functions. The first one sends a command to the LCD, by having the RS pin low. The second and third functions send data to be displayed on the LCD by having the RS pin high.

```rust
/// Send command to the LCD
///
/// # Arguments
///
/// * `cmd` - Command to send
pub fn send_cmd(&mut self, cmd: u8) {
    self.rs.set_low().ok();

    let higher_nibble = (cmd >> 4) & 0x0F;
    let lower_nibble = (cmd >> 0) & 0x0F;

    self._send_nibble(higher_nibble);
    self._send_nibble(lower_nibble);
}

/// Send data to the LCD
///
/// # Arguments
///
/// * `data` - Byte to send
#[allow(dead_code)]
pub fn send_data(&mut self, data: u8) {
    self.rs.set_high().ok();

    let higher_nibble = (data >> 4) & 0x0F;
    let lower_nibble = (data >> 0) & 0x0F;

    self._send_nibble(higher_nibble);
    self._send_nibble(lower_nibble);

    self.delay.delay_us(40_u16);
}

/// Send a string to the LCD
///
/// # Arguments
///
/// * `string` - String to send
#[allow(dead_code)]
pub fn send_string(&mut self, string: &str) {
    for byte in string.chars() {
        self.send_data(byte as u8);
    }
}
```

# Using the module  
In a ```main.rs``` file, in the same folder of the ```lcd.rs``` module, we can import the module and write a simple code where the LCD is initialized
and a **Hello World!** is printed.

```rust
// src/main.rs

// std and main are not available for bare metal software
#![no_std]
#![no_main]

mod lcd;

use lcd::LCD;
use panic_halt as _;

use cortex_m_rt::entry;
use hal::{delay::Delay, pac, prelude::*};
use stm32f1xx_hal as hal;

#[entry]
fn main() -> ! {
    /* Get access to device and core peripherals */
    let dp = pac::Peripherals::take().unwrap();
    let cp = cortex_m::Peripherals::take().unwrap();

    /* Get access to RCC, AFIO and GPIOA */
    let mut rcc = dp.RCC.constrain();
    let mut flash = dp.FLASH.constrain();
    let mut gpioa = dp.GPIOA.split(&mut rcc.apb2);
    let mut gpioc = dp.GPIOC.split(&mut rcc.apb2);

    // Freeze clocks
    let clocks = rcc.cfgr.freeze(&mut flash.acr);

    // Get delay instance
    let delay = Delay::new(cp.SYST, clocks);

    // Configure pin for LCD
    let rs = gpioa.pa0.into_push_pull_output(&mut gpioa.crl);
    let en = gpioa.pa1.into_push_pull_output(&mut gpioa.crl);
    let d4 = gpioc.pc0.into_push_pull_output(&mut gpioc.crl);
    let d5 = gpioc.pc1.into_push_pull_output(&mut gpioc.crl);
    let d6 = gpioc.pc2.into_push_pull_output(&mut gpioc.crl);
    let d7 = gpioc.pc3.into_push_pull_output(&mut gpioc.crl);

    let mut lcd = LCD::new(rs, en, d4, d5, d6, d7, delay);
    lcd.init();

    lcd.send_string("Hello World!");

    loop {}
}
```

# Afterword
You can clone the whole project, including the LCD module and the ```main``` code from this [repository](https://github.com/bacelarhenrique/rust-lcd-display){:target="_blank"}.
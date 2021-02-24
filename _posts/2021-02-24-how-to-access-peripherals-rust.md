---
title:  "How to Access Peripherals in Embedded Rust"
excerpt: A guide to accessing memory mapped registers for programming peripherals in Rust.
tags: rust embedded peripherals gpio stm32 led
header:
  teaser: assets/images/peripherals.jpg
---

# Preface
When working with microcontrollers, programming peripherals registers is like the basics of an embedded system. Peripherals are used to interact with systems outside of the microcontroller, as well as directly and indirectly interacting with their surroundings in the world via sensors, motor controllers, or human interfaces such as a display or keyboard.

In this article, I'll explain the basic knowledge necessary to access those memory regions in order to configure and use these peripherals in Rust without using a higher-layer library.

# Memory space
Although 32 bit microcontrollers have a real and linear address space from ```0x0000_0000``` to ```0xFFFF_FFFF```, only a few hundred kilobytes of that range is used for actual memory, leaving a significant amount of space remaining. However, rather than ignoring all remaining space, microcontroller designers instead map the interface for peripherals in certain memory locations.

This ends up looking like this:

<figure>
    <a href="/assets/images/nrf52-memory-map.png"><img src="/assets/images/nrf52-memory-map.png"></a>
    <figcaption><a href="http://infocenter.nordicsemi.com/pdf/nRF52832_PS_v1.1.pdf" target="_blank">Nordic nRF52832 Datasheet</a></figcaption>
</figure>

# Accessing the peripheral
For this example, I'll be accessing the GPIO's Port A from a STM32F103RB MCU in order to configure it as an output and blink an LED.

## Defining the registers
In Rust, a collection of registers can be represented using a ```struct``` - exactly the same way we do it in C.

```rust
#[repr(C)]
struct GPIOBlock {
    pub crl: u32,
    pub crh: u32,
    pub idr: u32,
    pub odr: u32,
    pub bsrr: u32,
    pub brr: u16,
    pub lckr: u32,
}
```

The qualifier ```#[repr(C)]``` is used make sure the Rust compiler will lay this structure out like a C compiler would. This means that the structure won't be re-ordered, as allowed by Rust but not by C. You can just imagine the problem we'd have debugging in an eventual re-ordering of the position of the registers in the memory.

With this structure, we have six registers of 32 bits and one of 16 bits, composing the set of registers of the GPIO peripheral. Now, we can create a variable to this structure pointing to the Port A address in the memory. 

```rust
let gpioa = 0x4001_0800 as *mut GPIOBlock;
let crl = unsafe { (*gpio).crl };
```

You can check all peripheral addresses in the product's [Reference Manual](https://www.st.com/resource/en/reference_manual/cd00171190-stm32f101xx-stm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf){:target="_blank"}.

In this case, there are a couple of problems with this approach:

1. We have to use unsafe every time we want to access our Peripheral.
2. We've got no way of specifying which registers are read-only or read-write.
3. Any piece of code anywhere in your program could access the hardware through this structure.
4. Most importantly, it doesn't actually work...

The problem is that compilers are clever. When accessing a piece of RAM twice, one after the other, the compiler will simply skip the first one entirely. In C, we can make sure every access occurs as intended by marking variables as ```volatile```. In Rust, we instead mark de *accesses* as ```volatile```, not the variable.

```rust
let gpioa = unsafe { &mut *(0x4001_0800 as *mut GPIOBlock) };
let crl = unsafe { core::ptr::read_volatile(&mut gpioa.crl) };
```

So, we fixed the problem but now we have even more ```unsafe``` code. Fortunately, there's a third party crate which can help us - ```volatile_register```. This library implements volatile access to memory mapped hardware registers.

```rust
use volatile_register::RW;

#[repr(C)]
struct GPIOBlock {
    pub crl: RW<u32>,
    pub crh: RW<u32>,
    pub idr: RW<u32>,
    pub odr: RW<u32>,
    pub bsrr: RW<u32>,
    pub brr: RW<u16>,
    pub lckr: RW<u32>,
}

fn get_gpioa() -> &'static mut GPIOBlock {
    unsafe { &mut *(0x4001_0800 as *mut GPIOBlock) }
}

fn get_crl() -> u32 {
    let gpioa = get_gpioa();
    gpioa.crl.read()
}
```

Now, volatile accesses are performed automatically through the ```read``` and ```write``` methods.

## The Rust wrapper
Finally, we can wrap our structure up into a higher-layer API that is safe for users to call.

```rust
use volatile_register::RW;

#[repr(C)]
struct GPIOBlock {
    pub crl: RW<u32>,  // Port configuration register low
    pub crh: RW<u32>,  // Port configuration register high
    pub idr: RW<u32>,  // Port input data register
    pub odr: RW<u32>,  // Port output data register
    pub bsrr: RW<u32>, // Port bit set/reset register
    pub brr: RW<u16>,  // Port bit reset register
    pub lckr: RW<u32>, // Port configuration lock register
}

/// GPIOA Struct
pub struct GPIOA {
    p: &'static mut GPIOBlock,
}

/// GPIOA Implementation
impl GPIOA {
    pub fn new() -> GPIOA {
        GPIOA {
            p: unsafe { &mut *(0x4001_0800 as *mut GPIOBlock) },
        }
    }

    /// Configures GPIOA as output push-pull
    pub fn into_push_pull_output(&mut self) {
        unsafe { self.p.crl.modify(|r| (r & 0x1101_1111) | 0x0010_0000) }
    }

    /// Sets pin
    pub fn set_pin(&mut self, pin: u8) {
        unsafe { self.p.bsrr.write(0x0000_0001 << (pin & 0xFF)) }
    }

    /// Clears pin
    pub fn clear_pin(&mut self, pin: u8) {
        unsafe { self.p.brr.write(0x0000_0001 << (pin & 0xFF)) }
    }
}
```

We can use this API in our ```main``` to configure the pin 5 of the Port A and finally blink our LED.

```rust
// src/main.rs

// std and main are not available for bare metal software
#![no_std]
#![no_main]

use panic_halt as _;

use cortex_m_rt::entry;
use hal::{delay::Delay, pac, prelude::*};
use stm32f1xx_hal as hal;

const LED_PIN: u8 = 5;

#[entry]
fn main() -> ! {
    /* Get access to device and core peripherals */
    let dp = pac::Peripherals::take().unwrap();
    let cp = cortex_m::Peripherals::take().unwrap();

    /* Get access to RCC, AFIO and GPIOA */
    let rcc = dp.RCC;
    let mut flash = dp.FLASH.constrain();
    let mut gpioa = GPIOA::new();

    /* Enable PORTA clock */
    rcc.apb2enr.write(|w| w.iopaen().enabled());

    /* Set up LED pin */
    gpioa.into_push_pull_output();

    /* Set up sysclk and freeze it */
    let clocks = rcc.constrain().cfgr.sysclk(8.mhz()).freeze(&mut flash.acr);

    /* Set up systick delay */
    let mut delay = Delay::new(cp.SYST, clocks);

    loop {
        gpioa.set_pin(LED_PIN);
        delay.delay_ms(1_000_u16);
        gpioa.clear_pin(LED_PIN);
        delay.delay_ms(1_000_u16);
    }
}
```

# References
- [The Embedded Rust book](https://docs.rust-embedded.org/book/){:target="_blank"}
- [```volatile_register``` docs](https://docs.rs/volatile-register/0.2.0/volatile_register/){:target="_blank"}
- [STM32F10x Reference Manual](https://www.st.com/resource/en/reference_manual/cd00171190-stm32f101xx-stm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf){:target="_blank"}
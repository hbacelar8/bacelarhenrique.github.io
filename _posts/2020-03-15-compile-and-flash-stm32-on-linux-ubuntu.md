---
title:  "Compile and flash STM32 on Linux (Ubuntu)"
tags: stm32 ubuntu
---

# Introduction
This tutorial shows how to set up the tools for compiling and flashing code into STM32 microcontroller on Ubuntu. We will be using the ARM GCC compiler toolchain and stlink utils.

The GNU Arm Embedded toolchain contains integrated and validated packages featuring the Arm Embedded GCC compiler, libraries and other GNU tools necessary for bare-metal software development on devices based on the Arm Cortex-M and Cortex-R processors. The toolchains are available for cross-compilation on Microsoft Windows, Linux and Mac OS X host operating systems.

# Installation
Both ```gcc-arm-none-eabi``` compiler and ```stlink utils``` can be installed via terminal.

## Installing gcc-arm-none-eabi
```bash
sudo apt install gcc-arm-none-eabi
```

## Installing stlink utils
```bash
sudo apt install st-utils
```

# Compiling and Flashing
Now that both tools have been installed, the next step is to compile the source code into a ```.elf``` file, so we can generate the ```.bin``` file and finally flash this binary into the STM32 chip.

For instance, to compile a code in a ```main.c``` file into a Cortex M0 microcontroller, just type the following inside the file's folder

```bash
arm-none-eabi-gcc -std=gnu99 -g -O2 -Wall -mlittle-endian -mthumb -mthumb-interwork -mcpu=cortex-m0 -fsingle-precision-constant -Wdouble-promotion main.c -o main.elf
```

Most programmers will not accept a GNU executable as an input file, so we need to convert the information from the .elf file into a .bin binary file. To do so, we'll use the GNU utility called arm-none-eabi-objcopy.

```bash
arm-none-eabi-objcopy -O binary main.elf main.bin
```

Finally, to flash the binary file, with the board connected via USB, just type on the terminal

```bash
st-flash write main.bin 0x8000000
```

This will flash the the binary file starting from the memory position ```0x8000000``` (this should be adapted to your device's memory mapping).

# Makefile
To make the whole process easier, we can automate it with the use of a Makefile. By configuring it with all the parameters we've used before, we can do everything simply by executing it with the ```make``` command on the terminal. More information about GNU Make can be found on it's [website](https://www.gnu.org/software/make/manual/make.html).

With the Makefile and the ```main.c``` file in the same folder, just type the command ```make``` on the terminal and all the compilation will be done. After that, to flash the code, simply type

```bash
make flash
```

The following command can be used to remove the ```.o```, ```.elf``` and ```.bin``` files generated during the compilation

```bash
make clean
```

# Conclusion
In this tutorial we've seen how to compile and burn a source code into the STM32 microcontroller using the ARM GNU compiler and the stlink utils package on Ubuntu. We've also seen how to automate the whole process using a Makefile, which can be used in the future to a more complex project with all its source files and includes.
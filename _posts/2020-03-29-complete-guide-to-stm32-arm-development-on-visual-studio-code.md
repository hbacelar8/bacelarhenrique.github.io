---
title:  "Complete Guide to STM32 ARM Development on Visual Studio Code"
tags: stm32 vscode
---

# Introduction
Visual Studio Code was a great contribution for the coding world. From thousands of plugins, themes and its great IntelliSense, it is a great cross-platform code editor for working with. However, for embedded projects for microcontrollers like those generated for STM32 ARM Cortex MCUs, it's important to know everything that must be configured to the development and debugging of a project.

Therefore, in this article, I'll explain all the steps required for this matter. I'm going to assume that you already have your code structured and a Makefile ready, generated whether by the [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html) initialisation code generator or downloaded from your MCU's respective website. More information on the official [website](https://www.st.com/en/microcontrollers-microprocessors/stm32-32-bit-arm-cortex-mcus.html).

In this tutorial, I'll be using the [STM32 Nucleo-F101RB](https://www.st.com/en/evaluation-tools/nucleo-f103rb.html) development board with a [STM32F103RB](https://web.archive.org/web/20200807005833/https://www.st.com/en/microcontrollers-microprocessors/stm32f101rb.html) microcontroller. I'll also be working on Windows, which requires some additional steps to set things up. All the differences between Linux and Windows will be addressed throughout the article.

# Installations
Everything that must be installed is listed below.

- Visual Studio Code
- C/C++ Plugin
- Cortex Debug Plugin
- ARM GCC Compiler
- STM32 ST-LINK Utility
- MinGW / MSYS2 (Windows)
- OpenOCD

## Visual Studio Code
The protagonist of this story, Visual Studio Code (from now on called vscode) can be downloaded [here](https://code.visualstudio.com/).

## Plugins
After installing vscode, open it and open the Extensions by clicking on its icon on the left bar or by typing `Ctrl+Shift+X`. Search for C/C++ and install it, or download it from vscode's marketplace [here](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools). This plugin offers language support for C/C++ for vscode, including features such as IntelliSense and debugging.

However, we'll be using the Cortex-Debug plugin for debugging. On the same Extensions bar, search for Cortex-Debug and install it. You can also download it from the market place [here](https://marketplace.visualstudio.com/items?itemName=marus25.cortex-debug).

## ARM GCC Compiler
Download the ARM compiler [here](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads) and install it. During installation, some necessary drivers will be installed. In the end, don't forget to select the option to add it to the PATH variable.

For Linux, I recommend you to download it from the same website linked above for Windows since the recent repository version does not come with the arm-none-eabi-gdb debugger program which will be necessary for debugging.

## ST-LINK Utility
STM32 ST-LINK Utility is a full-featured software interface for programming STM32 microcontrollers. It provides an easy-to-use and efficient environment for reading, writing and verifying a memory device. It will be especially necessary to program the chip.

Basically, to install it on Ubuntu, just type on your terminal

```bash
sudo apt install st-utils
```

You can download the Windows version [here](https://www.st.com/en/development-tools/stsw-link004.html). It consists of a graphical user interface (GUI) with a command line interface (CLI).

I'll come back on how to prepare your Makefile later on this article.

## MinGW / MSYS2
MinGW is a minimalist development environment for native Windows applications. It will be necessary if you are using Windows like me, since it provides a GCC compiler and more importantly a command line interpreter system (MSYS), which will be useful to compile Makefiles. Download the most recent version [here](https://osdn.net/projects/mingw/releases/). After installing it, the MinGW Installation Manager will open, and you'll be able to select the packages you want to install. Select both packages as shown in the figure below.

Click on **Installation** on the menu bar on the top of the window and then **Apply Changes**.

In order to debug your code, you'll need the software OpenOCD. To install it on Windows, MSYS2 is necessary. MSYS2 is, at its core, an independent rewrite of MSYS, based on modern Cygwin and MinGW-w64. It features the package management system Pacman, for providing easy installation of packages. Download it [here](https://www.msys2.org/) and install it.

After closing the installer, the MSYS2 terminal will open. Run the following command multiple times until there are no more updates. You'll need to close the terminal. To reopen it, you can find it in its installation folder at ```C:\msys64\msys2.exe```.

```bash
pacman -Syuu
```

## OpenOCD
Finally, the last software to install is OpenOCD. The Open On-Chip Debugger is essential for debugging our code. Installing it on Ubuntu is as simple as typing on your terminal

```bash
sudo apt install openocd
```

For Windows, open the MSYS2 terminal and type

```bash
pacman -S mingw-w64-x86_64-toolchain
```

# Include Paths and Defines
For this section I'll explain how to configure the paths and defines of the project on vscode. Below, you can see the structure of my project folder. Again, the code was generated with the STM32CubeMX software.

Open your project folder on vscode. By opening a file, like your ```main.c```, you'll notice that there will be a lot of include errors. By default, vscode will search recursively over all the folders inside the one you opened your project. Since all API drivers files are inside those folders, everything should've worked just fine. However, some defines must be passed to C/C++ plugin's configuration. Those defines can be found inside your Makefile, since they're also passed to the compiler.

Open C/C++ plugin's configuration by pressing ```Ctrl+Shift+P``` and searching for **Edit Configurations (JSON)**. A JSON file called ```c_cpp_properties.json``` will open inside the new folder called **.vscode** created on your project's root.

Open your Makefile and look for a variable called **C_DEFS** like shown below.

```make
# C defines
C_DEFS =  \
-DUSE_FULL_LL_DRIVER \
-DHSE_VALUE=8000000 \
-DHSE_STARTUP_TIMEOUT=100 \
-DLSE_STARTUP_TIMEOUT=5000 \
-DLSE_VALUE=32768 \
-DHSI_VALUE=8000000 \
-DLSI_VALUE=40000 \
-DVDD_VALUE=3300 \
-DPREFETCH_ENABLE=1 \
-DSTM32F103xB
```

We are interested in two defines here: the first and the last one. The first one tells the compiler that we are using the low level API library. The last one indicates the microcontroller we're using. If you chose to generate your code using the HAL library for drivers, the first will mention HAL instead.

Back in the JSON configuration file, remove everything that is currently in the defines tag and copy the ones that we'll use without the -D. Your file should look like this

```json
{
    "configurations": [
    {
        "name": "Win32",
        "includePath": [
        "${workspaceFolder}/**"
        ],
        "defines": [
        "USE_FULL_LL_DRIVER",
        "STM32F103xB"
        ],
        "cStandard": "c11",
        "intelliSenseMode": "msvc-x64"
    }
    ],
    "version": 4
}
```

Close vscode and reopen it. Everything should now be fine and all include errors are gone.

# Compiling and Flashing
Basically, if you're using Ubuntu, just add the following lines to the end of your Makefile

```make
flash:
    st-flash write $(BUILD_DIR)/$(TARGET).bin 0x8000000
```

And for Windows

```make
flash:
    st-link_cli -me
    st-link_cli -P $(BUILD_DIR)/$(TARGET).bin 0x8000000 -V "after_programming"
```

Now, after compiling, you can flash the code by simple typing ```make flash``` on the terminal on the project's root. This will erase the flash memory and program the chip from the memory position ```0x8000000```.

# Debugging
To configure the debugging, open the debug tab on the left bar on vscode or type ```Ctrl+Shift+D```. Since there is no configuration yet, we'll customize **Run and Debug** by creating a ```launch.json``` file. After clicking on this option, you'll see some environments to choose. Select Cortex-Debug. The file will be created in the .vscode folder and opened.

The configuration varies depending on the microcontroller and board you're using. As I mentioned before, I'm using the STM32 Nucleo-F101RB development board with a STM32F103RB microcontroller. My configuration is as follows

It's important to change the **executable** tag to point to your project's ```.elf``` file. The **servertype** tag must be configured as **openocd** and the **device** and **configFiles** according to your microcontroller and board. You can look for more information on the Cortex-Debug plugin's [website](https://marcelball.ca/projects/cortex-debug/cortex-debug-launch-configurations/).

It's important to note that what's in the **configFiles** tag is the path to the ```.cfg``` file that will be passed to OpenOCD. With OpenOCD installation, some config files from some generic boards are also installed, but you can always create your own. You can have more information on [OpenOCD's Guide Line](http://openocd.org/doc-release/pdf/openocd.pdf).

Now you can finally head back to the Run and Debug tab and start debugging. If you get and OpenOCD error, it's probably because whether the debugger can't find your OpenOCD executable or your ARM GDB path. To fix it, create a file called ```settings.json``` inside your **.vscode** folder (if it doesn't already exist) with the following instructions to the Cortex-Debug plugin, adjusting the paths if it's the case

```json
{
    "cortex-debug.armToolchainPath": "C:/Program Files (x86)/GNU Tools Arm Embedded/9 2019-q4-major/bin",
    "cortex-debug.openocdPath": "C:/msys64/mingw64/bin/openocd.exe"
}
```

# Conclusion
Well, that's it. I think I've explained all the steps required to work with STM32 (compile and debug) using Visual Studio Code, whether you're using Linux or Windows. We have seen how to install all the softwares required, set up the Makefile to compile and flash the code and finally how to debug.

This has been a long article to write and I hope I have covered everything in a clear and understandable way. If you have any problems with the whole process or something to add, let me know through the comments section below.
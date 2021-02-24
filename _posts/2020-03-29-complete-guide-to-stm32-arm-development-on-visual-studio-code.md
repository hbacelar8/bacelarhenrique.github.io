---
title:  "Complete Guide to STM32 ARM Development on Visual Studio Code"
excerpt: A guide to programming and debugging embedded projects using STM32 ARM Cortex MCUs on Visual Studio Code.
tags: stm32 vscode ubuntu debugging embedded
header:
  teaser: assets/images/vscode-c.jpg
---

# Introduction
Visual Studio Code was a great contribution for the coding world. From thousands of plugins, themes and its great IntelliSense, it is a great cross-platform code editor for working with. However, for embedded projects for microcontrollers like those generated for STM32 ARM Cortex MCUs, it's important to know everything that must be configured to the development and debugging of a project.

Therefore, in this article, I'll explain all the steps required for this matter. I'm going to assume that you already have your code structured and a Makefile ready, generated whether by the [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html){:target="_blank"} initialisation code generator or downloaded from your MCU's respective website. More information on the official [website](https://www.st.com/en/microcontrollers-microprocessors/stm32-32-bit-arm-cortex-mcus.html){:target="_blank"}.

In this tutorial, I'll be using the [STM32 Nucleo-F101RB](https://www.st.com/en/evaluation-tools/nucleo-f103rb.html){:target="_blank"} development board with a [STM32F103RB](https://www.st.com/en/microcontrollers-microprocessors/stm32f101rb.html){:target="_blank"} microcontroller. I'll be working on Linux (Ubuntu).

# Installations
Everything that must be installed is listed below.

- Visual Studio Code
- VSCode C/C++ Plugin
- VSCode Cortex Debug Plugin
- GNU Arm Embedded Toolchain
- STM32 ST-LINK Utility
- OpenOCD

## Visual Studio Code
The protagonist of this story, Visual Studio Code (from now on called vscode) can be downloaded [here](https://code.visualstudio.com/){:target="_blank"}.

## Plugins
After installing vscode, open it and open the Extensions tab by clicking on its icon on the left bar or by typing `Ctrl+Shift+X`. Search for C/C++ and install it, or download it from vscode's marketplace [here](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools){:target="_blank"}. This plugin offers language support for C/C++ for vscode, including features such as IntelliSense and debugging.

However, we'll be using the Cortex-Debug plugin for debugging. On the same Extensions bar, search for Cortex-Debug and install it. You can also download it from the market place [here](https://marketplace.visualstudio.com/items?itemName=marus25.cortex-debug){:target="_blank"}.

## GNU Arm Embedded Toolchain
The GNU Arm Embedded toolchain can be downloaded from Ubuntu's package manager.

```bash
sudo apt install gcc-arm-none-eabi
```

For debugging, we'll use the GNU Debugger that can be installed by with

```bash
sudo apt install gdb-multiarch
```

## ST-LINK Utility
STM32 ST-LINK Utility is a full-featured software interface for programming STM32 microcontrollers. It provides an easy-to-use and efficient environment for reading, writing and verifying a memory device. It will be especially necessary to program the chip.

Basically, to install it on Ubuntu, just type on your terminal

```bash
sudo apt install st-utils
```

## OpenOCD
Finally, the last software to install is OpenOCD. The Open On-Chip Debugger is essential for debugging our code. Installing it on Ubuntu is as simple as typing on your terminal

```bash
sudo apt install openocd
```

# Include Paths and Defines
For this section I'll explain how to configure the paths and defines of the project on vscode.

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

Back in the JSON configuration file, remove everything that is currently in the defines tag and copy the ones that we'll use without the ```-D```. Your file should look like this

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
Basically, just add the following lines to the end of your Makefile

```make
flash:
    st-flash write $(BUILD_DIR)/$(TARGET).bin 0x8000000
```

Now, after compiling, you can flash the code by simple typing ```make flash``` on the terminal on the project's root. This will erase the flash memory and program the chip from the memory position ```0x8000000```.

# Debugging
To configure the debugging, open the debug tab on the left bar on vscode or type ```Ctrl+Shift+D```. Since there is no configuration yet, we'll customize **Run and Debug** by creating a ```launch.json``` file. After clicking on this option, you'll see some environments to choose. Select Cortex-Debug. The file will be created in the .vscode folder and opened.

The configuration varies depending on the microcontroller and board you're using. As I mentioned before, I'm using the STM32 Nucleo-F101RB development board with a STM32F103RB microcontroller. My configuration is as follows

```bash
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Cortex Debug",
            "cwd": "${workspaceRoot}",
            "executable": "PATH_TO_GENERATED_ELF_FILE",
            "request": "launch",
            "type": "cortex-debug",
            "servertype": "openocd",
            "device": "STM32F103RB",
            "configFiles": [
                "board/st_nucleo_f103rb.cfg"
            ]
        }
    ]
}
```

It's important to change the **executable** tag to point to your project's ```.elf``` file. The **servertype** tag must be configured as **openocd** and the **device** and **configFiles** according to your microcontroller and board. You can look for more information on the Cortex-Debug plugin's [website](https://marcelball.ca/projects/cortex-debug/cortex-debug-launch-configurations/){:target="_blank"}.

It's important to note that what's in the **configFiles** tag is the path to the ```.cfg``` file that will be passed to OpenOCD. With OpenOCD installation, some config files from some generic boards are also installed, but you can always create your own. You can have more information on [OpenOCD's Guide Line](http://openocd.org/doc-release/pdf/openocd.pdf){:target="_blank"}. You can check the available ```.cfg``` files that came with **openocd** installation in the folder ```/usr/share/openocd```.

Now, if you try to start debugging, you'll get an error since the plugin Cortex Debug will try to run the ```arm-none-eabi-gdb``` which no longer comes with the recent versions of the GNU ARM Embedded toolchain available on Ubuntu's repository. That's why we installed the ```gdb-multiarch```. In that case, we can create the following *symlink* to redirect the ```arm-none-eabi-gdb``` command to the ```gdb-multiarch``` binary.

```bash
ln -s /usr/bin/gdb-multiarch /usr/bin/arm-none-eabi-gdb
```

Now you can finally head back to the Run and Debug tab and start debugging.

# Conclusion
Well, that's it. I think I've explained all the steps required to work with STM32 (compile and debug) using Visual Studio Code on Ubuntu. We have seen how to install all the softwares required, set up the Makefile to compile and flash the code and finally how to debug.

This has been a long article to write and I hope I have covered everything in a clear and understandable way. If you have any problems with the whole process or something to add, let me know through the comments section below.
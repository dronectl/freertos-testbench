# Installation Guide

Modified: 2023-01

Required software installation guide.

## Navigation
1. [Toolchain Installation](#toolchain-installation)
2. [OpenOCD Installation](#openocd-installation)
3. [CMake Installation](#cmake-installation)

> Note for Windows Installations - The binary files included as part of these software installations should be included as part of your system path. If the installer has the option, be sure to enable system path discovery.

## Toolchain Installation
This demo version 10.3 which at the time of writing is the latest arm toolchain. Note that these downloads are large and may take some time.
### MacOSX
Get the CMSIS `arm-none-eabi-gcc` toolchain for ARM Cortex microcontrollers:
```bash
brew install --cask gcc-arm-embedded
```
### Windows
Download version 10.3 from the windows install link [here](https://developer.arm.com/downloads/-/gnu-rm)

## OpenOCD Installation
This demo version 0.11.0 which at the time of writing is the latest `openocd` version
### MacOSX
Get `openocd` using `brew`:
```bash
brew install openocd
```
### Windows
Download xPack OpenOCD v0.11.0-5 (xpack-openocd-0.11.0-5-win32-x64.zip) from the github release link [here](https://github.com/xpack-dev-tools/openocd-xpack/releases)

## CMake Installation
This demo requires a minimum `cmake` version 3.24 to build.
### MacOSX
Install `cmake` using `brew`:
```bash
brew install cmake
```
### Windows
Download version `3.34` or higher from the windows install link [here](https://cmake.org/download/)
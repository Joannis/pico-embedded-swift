# Embedded Swift on the Raspberry Pi Pico

A proof of concept for executing Embedded Swift code on the [Raspberry Pi Pico](https://www.raspberrypi.com/products/raspberry-pi-pico/). The main program is written in C (built with the official Pico C SDK) and calls into a statically linked Swift library.

## Prerequisites and Installation

### Hardware

- A [Raspberry Pi Pico](https://www.raspberrypi.com/products/raspberry-pi-pico/)
- Optional but recommended: a [Raspberry Pi Debug Probe](https://www.raspberrypi.com/products/debug-probe/) for more convenient flashing (see below). This also enables debugging.

### Software

- Host OS: macOS 13.x or 14.x
  
  Tested on macOS 13.6.2 and macOS 14.0. It’ll probably work on Linux with minimal modifications to tell CMake how to find the Swift toolchain, but I haven’t tested this.

- A recent nightly Swift toolchain from [swift.org](https://www.swift.org/download/). Tested with the Xcode toolchain from December 7, 2023 from the Trunk Development (main) Snapshots.

- A clone of the [Raspberry Pi Pico C/C++ SDK](https://github.com/raspberrypi/pico-sdk/):

  ```sh
  cd ..
  git clone https://github.com/raspberrypi/pico-sdk.git
  cd pico-sdk/
  git submodule update --init
  cd ..
  ```

  This project expects to find the SDK in a sibling directory named `pico-sdk`. You can change this below if your SDK is in a different place.

- The GCC toolchain for ARM embedded platforms (the Pico SDK builds with GCC by default):

  ```sh
  brew install --cask gcc-arm-embedded
  ```

- [CMake](https://cmake.org/) and [Ninja](https://ninja-build.org/):

  ```sh
  brew install cmake --HEAD
  brew install ninja
  ```

  The Pico SDK uses CMake (as for 01/2024 we need recent patches from HEAD branch) as its build system and we’re piggybacking on that. And CMake’s Swift support only works with Ninja. The Swift library is also built with CMake. The unfortunate consequence is that we can’t easily use a [SwiftPM](https://www.swift.org/package-manager/) package for the Swift library as we’d have to tell CMake how to build the package.

## Configuration

Open the file `CMakeLists.txt` in the root folder. Edit these two lines to match your setup:

```cmake
…
set(PICO_SDK_PATH "${CMAKE_CURRENT_LIST_DIR}/../pico-sdk")
…
set(Swift_Toolchain "org.swift.59202312071a")
…
```

## Building

```sh
cd pico-embedded-swift
CMAKE_EXPORT_COMPILE_COMMANDS=1 cmake -GNinja -Bbuild
ninja -Cbuild
```

This produces the executable `SwiftPico.elf` in the `build` directory.

Note that `CMAKE_EXPORT_COMPILE_COMMANDS=1` is mandatory if you want VSCode to have IDE support for the project. It tells CMake to generate a file named `compile_commands.json` in the build directory. VSCode will pick this up automatically and use it to provide code completion and other IDE features.

## Troubleshooting

1. Cmake error `<unknown>:0: error: unable to load standard library for target 'armv6m-none-none-eabi'`
if you see this error, it means that you used Cmake without the Swift embeedded support. Upgrade your Cmake to the latest version (`brew install cmake --HEAD`).

2. `ninja: error: 'CMakeFiles/SwiftLib.dir/SwiftLib.swift.obj', needed by 'SwiftLib' missing and no known rule to make it`
You used Swift toolchain, which doesn't have embedded target support (`-enable-experimental-feature Embeded`). Use the newest Swift compilation from `main`. For example, the latest Xcode toolchain from December 7, 2023 from the Trunk Development (main) Snapshots works fine.

## Running on the Pico

You have two options to copy the executable to the Pico:

1. Via the USB Mass Storage interface:

    - Connect the Pico to your computer while holding down the BOOTSEL button. When you release the button, the Pico will appear as a removable drive.
 
    - Use the `elf2uf2` tool (conveniently included in the build directory) to convert the ELF file to the required UF2 format and copy it to the Pico:

      ```sh
      ./elf2uf2/elf2uf2 SwiftPico.elf /Volumes/RPI-RP2/SwiftPico.uf2
      ```
    
    - The Pico will automatically reboot and run the program (you can ignore macOS’s “disk not ejected properly” message).

2. Via the debug probe or another Pico device acting as a debugger (through the [Pico Probe software](https://github.com/raspberrypi/picoprobe)). The debug probe is connected to your PC and talks to the Pico via its debug port. This allows you to reflash the Pico without having to disconnect it.

    I use [probe.rs](https://probe.rs/) for this, which is a tool from the Rust community, but it works in this context too. Provided you have [Rust](https://www.rust-lang.org/) installed, you can install probe-rs with `cargo install probe-rs-debugger --features cli`.

    ```sh
    probe-rs run --chip RP2040 SwiftPico.elf
    ```

### Code Completion

You'll need to build the latest version of CMake for this to work. For this project, [this PR by Evan Wilde](https://gitlab.kitware.com/cmake/cmake/-/merge_requests/9095) was used. Also ensure that VSCode is configured to use the `swift` toolchain configured above. Also configure the VSCode extension to use `sourcekit-lsp` within this Swift toolchain.

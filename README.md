# OpenTTD-on-Raspberry

This repository provides the instructions to compile OpenTTD from source on a PC to use on a Raspberry Pi.
This cross-compiling is needed because the compilation process exceeds the memory available on older models.

**Note**: The instructions have the goal to produce the necessary files for running an OpenTTD server on a Raspberry Pi.
To create a version with graphics and UI that can be played on the Pi itself, refer to the compilation instructions from OpenTTD to alter the commands and options used in this guide.

## References

OpenTTD compiling instructions: https://github.com/OpenTTD/OpenTTD/blob/master/COMPILING.md \
Raspberry Pi GCC Toolchains: https://sourceforge.net/projects/raspberry-pi-cross-compilers/ \
Raspberry Pi GCC Toolchains Github: https://github.com/abhiTronix/raspberry-pi-cross-compilers \
Raspberry Pi GCC Cross-Compiling instructions: https://github.com/abhiTronix/raspberry-pi-cross-compilers/wiki/Cross-Compiler:-Installation-Instructions \
OpenTTD Server manual: https://wiki.openttd.org/en/Manual/Dedicated%20server

## Instructions

Almost all instructions need to be carried out on a normal Linux system with enough RAM. This can also be a Linux VM or the WSL.

### Gathering sources, tools and files

#### OpenTTD Sources
To get the source files of OpenTTD go to https://www.openttd.org/downloads/openttd-releases/latest and scroll down to **Developer Files**. Download one of the sources archives and extract them to a known location. This location will be referenced as **Source Directory**.

#### Cross-Compiler Toolchain
Next get the cross-compiler toolchain from https://sourceforge.net/projects/raspberry-pi-cross-compilers/files/Raspberry%20Pi%20GCC%20Cross-Compiler%20Toolchains/ . Choose the Debian version of the Raspberry OS you want to use on your Raspberry Pi. Refer to https://www.raspberrypi.com/software/operating-systems/ to find out the version name. Next choose the compiler version, normally you want to use the highest/latest. Then select your Raspberry Pi model and download the archive containing the compiler toolchain. Extract the archive to a known location that will be referenced as **Compiler Directory**. All libraries required for the compilation are available in a standard Ubuntu 22.04 installation. If you're using a different distro, please refer to the Raspberry Pi GCC Cross-Compiling instructions for the list of required packages to install.

#### rootfs
In order to be able to compile for the target OS, you need parts the root file system containing libraries that are used by programs like OpenTTD. Note that Raspberry Pi OS Desktop Bullseye/Debian 11 contains allmost all required libraries for OpenTTD. If you use a different version, make sure to install the required libraries listed in the OpenTTD compiling instructions.

First create a folder named like "rootfs" where you can store these files. It will be referred to as **rootfs Directory**. Now choose one of the methods to acquire the required files:

**a) Copy from installed Raspberry Pi** \
If you have your Raspberry Pi up and running with Raspberry Pi OS you can transfer these files over the network to your machine using the following commands. You need to change the IP to the local IP of your Pi and rootfs to the path of your **rootfs Directory**.

1. `rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.47:/lib rootfs`
2. `rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.47:/usr/include rootfs/usr`
3. `rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.47:/usr/lib rootfs/usr`

**b) Copy from Raspberry Pi OS image** \
If you don't have your Raspberry Pi up yet or can't transfer the files for other reasons, you can extract the files from the OS install image. First download the archived image from the Raspberry Pi OS website and extract it so that you have a .img file. Now mount this image as a virtual drive. In Ubuntu, you can open the Disks app and select the mounting option in the menu which will ask you for the .img file. You should then have two new drives, one named rootfs. You can get the path of this drive by opening a terminal via the file explorer. It will be something like `/media/user/rootfs`. Now execute the following commands with your paths to copy the files to your **rootfs Directory**.

1. `sudo rsync -avz  /media/user/rootfs/lib rootfs`
2. `sudo rsync -avz  /media/user/rootfs/usr/include rootfs/usr`
3. `sudo rsync -avz  /media/user/rootfs/usr/lib rootfs/usr`

After transferring the files via any methods there are still symbolic links left in these files that need to be fixed to point to your **rootfs Directory**. For that download the following python script by [abhiTronix](https://github.com/abhiTronix/raspberry-pi-cross-compilers): 
```
wget https://raw.githubusercontent.com/abhiTronix/rpi_rootfs/master/scripts/sysroot-relativelinks.py
```
And execute it after setting the execution rights: 
```
sudo chmod +x sysroot-relativelinks.py
./sysroot-relativelinks.py rootfs
```
There are also symbolic links missing that the compiler needs. For that download another script by [abhiTronix](https://github.com/abhiTronix/raspberry-pi-cross-compilers):
```
wget https://raw.githubusercontent.com/abhiTronix/raspberry-pi-cross-compilers/master/utils/SSymlinker
```
And execute the following commands:
```
sudo chmod +x SSymlinker
./SSymlinker -s rootfs/usr/include/arm-linux-gnueabihf/asm -d rootfs/usr/include
./SSymlinker -s rootfs/usr/include/arm-linux-gnueabihf/gnu -d rootfs/usr/include
./SSymlinker -s rootfs/usr/include/arm-linux-gnueabihf/bits -d rootfs/usr/include
./SSymlinker -s rootfs/usr/include/arm-linux-gnueabihf/sys -d rootfs/usr/include
./SSymlinker -s rootfs/usr/include/arm-linux-gnueabihf/openssl -d rootfs/usr/include
./SSymlinker -s rootfs/usr/lib/arm-linux-gnueabihf/crtn.o -d rootfs/usr/lib/crtn.o
./SSymlinker -s rootfs/usr/lib/arm-linux-gnueabihf/crt1.o -d rootfs/usr/lib/crt1.o
./SSymlinker -s rootfs/usr/lib/arm-linux-gnueabihf/crti.o -d rootfs/usr/lib/crti.o
```

#### Cross-compiling make file
To instruct CMake to cross-compile for the Raspberry Pi architecture a special make file is needed. You can use the following provided by [abhiTronix](https://github.com/abhiTronix/raspberry-pi-cross-compilers) and replace the two paths to your **Compiler Directory** and your **rootfs Directory**. Note that you can only use **absolute paths** in this file!
```
set(CMAKE_VERBOSE_MAKEFILE ON)
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_VERSION 1)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(tools <Compiler Directory>) # warning change toolchain path here.
set(rootfs_dir <rootfs Directory>) # warning change rootfs path here.

set(CMAKE_FIND_ROOT_PATH ${rootfs_dir})
set(CMAKE_SYSROOT ${rootfs_dir})

set(CMAKE_LIBRARY_ARCHITECTURE arm-linux-gnueabihf)
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fPIC -Wl,-rpath-link,${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} -L${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE}")
set(CMAKE_C_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wl,-rpath-link,${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} -L${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wl,-rpath-link,${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE} -L${CMAKE_SYSROOT}/usr/lib/${CMAKE_LIBRARY_ARCHITECTURE}")

#### Compiler Binary 
SET(BIN_PREFIX ${tools}/bin/arm-linux-gnueabihf)

SET (CMAKE_C_COMPILER ${BIN_PREFIX}-gcc)
SET (CMAKE_CXX_COMPILER ${BIN_PREFIX}-g++ )
SET (CMAKE_LINKER ${BIN_PREFIX}-ld 
            CACHE STRING "Set the cross-compiler tool LD" FORCE)
SET (CMAKE_AR ${BIN_PREFIX}-ar 
            CACHE STRING "Set the cross-compiler tool AR" FORCE)
SET (CMAKE_NM {BIN_PREFIX}-nm 
            CACHE STRING "Set the cross-compiler tool NM" FORCE)
SET (CMAKE_OBJCOPY ${BIN_PREFIX}-objcopy 
            CACHE STRING "Set the cross-compiler tool OBJCOPY" FORCE)
SET (CMAKE_OBJDUMP ${BIN_PREFIX}-objdump 
            CACHE STRING "Set the cross-compiler tool OBJDUMP" FORCE)
SET (CMAKE_RANLIB ${BIN_PREFIX}-ranlib 
            CACHE STRING "Set the cross-compiler tool RANLIB" FORCE)
SET (CMAKE_STRIP {BIN_PREFIX}-strip 
            CACHE STRING "Set the cross-compiler tool RANLIB" FORCE)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```
Place this file at a known location with a name like `PI.make` and rember it as **Toolchain File**.

#### liblzma
OpenTTD uses the library liblzma for the compression of save games. Unless you never want to save or load a game to/from file, you need this library. Unfortunately, the library binaries provided in the Debian packages for Raspberry Pi OS Bullseye are not usable to compile OpenTTD. Therefore you first need to cross-compile the library from sources. You can download the sources from https://tukaani.org/xz/. Extract the archive and get a terminal in the extracted directory. Next you need to find out the architecture name of the OS, which depends on the Raspberry Pi model. For example, the name for Raspberry OS on Raspberry Pi Model 2/3 is `arm-linux-gnueabihf`. Then execute the following commands one by one while replacing the paths to your **Compiler Directory** and **roofs Directory** and the OS/architecture:
```
PATH=<Compiler Directory>/bin:$PATH
LD_LIBRARY_PATH=<Compiler Directory>/lib:$LD_LIBRARY_PATH
./configure --target=arm-linux-gnueabihf --host=arm-linux-gnueabihf --prefix=<rootfs Directory>
make
sudo make install
```
The required library files should then be compiled and copied into your **rootfs Directory**.

### Compiling OpenTTD
The preparations for this step require much more effort than this actual step, so you're almost finished. Now get a terminal in your **Source Directory**. Because OpenTTD requires it's utilities to be available in the compilation process, you first need to compile them for your host machine. Execute the follwing commands in your **Source Directory**:
```
mkdir build-native
cd build-native
cmake .. -DCMAKE_BUILD_TYPE=RelWithdebInfo -DOPTION_DEDICATED=ON -DOPTION_TOOLS_ONLY=ON
make
```
The tools should now be compiled for your host machine. Now execute the following commands to compile OpenTTD for your Raspberry Pi while replacing the path to your **Toolchain File**:
```
cd ..
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=RelWithdebInfo -DOPTION_DEDICATED=ON -DCMAKE_TOOLCHAIN_FILE=<Toolchain File> -DOPTION_TOOLS_ONLY=OFF -DHOST_BINARY_DIR=../build-native
make
```
You're done! If you didn't get any errors to this point, OpenTTD should be compiled successfully for your Raspberry Pi. Now you can copy the following files and folders (or archive them first) from the build directory to your Raspberry Pi and start playing:
- openttd (file)
- ai (dir)
- baseset (dir)
- game (dir)
- lang (dir)
- scripts (dir)

## Disclaimer
I have build OpenTTD using these commands and tools for a friend for personal use. Therefore I can't garantuee for the operability or completeness of this guide. I also won't provide any personal support nor compile the binaries for all Raspberry Pi models and Raspberry Pi OS combinations. I am nevertheless open for any feedback or helpful tips (e.g. for another model/os combination) that will help others to be included in the guide. Also I am very thankful for the work of the people behind the tools and projects referenced here, because otherwise this would not have been possible.

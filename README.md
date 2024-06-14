# Booting Raspberry Pi 4b with custom Boot Protocol and Minimal Root File System 

## Table of Contents
1. [Hardware Requirements](#hardware-requirements)
2. [Preparation](#preparation)
3. [Create the Bootable SD Card](#create-the-bootable-sd-card)
4. [Create the Toolchain](#create-the-toolchain)
5. [Build the Bootloader](#build-the-bootloader)
6. [Build the Kernel](#build-the-kernel)
7. [Build the Root Filesystem](#build-the-root-filesystem)
8. [Preparing the Root Partition](#preparing-the-root-partition)
9. [Boot](#boot)

## Hardware Requirements

1. A Linux Desktop machine running Ubuntu to act as a host to cross compile source code. 
2. SD card and reader.

## Preparation

    Install required dependencies: 

    ```bash
    sudo apt install bc bison flex libssl-dev make libc6-dev libncurses5-dev
    ```

## Create the Bootable SD Card

1. Insert the SD card and find its device name:

    ```bash
    lsblk
    ```

    In this case, the device name is sdb and it already has two partitions sdb1 and sdb2. 

2. Deleting existing partitions, just to be sure:

    ```bash
    $ sudo fdisk /dev/sdb

    Welcome to fdisk (util-linux 2.34).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.


    Command (m for help): d
    Partition number (1,2, default 2):

    Partition 2 has been deleted.

    Command (m for help): d
    Selected partition 1
    Partition 1 has been deleted.

    Command (m for help): p
    Disk /dev/sdb: 1.85 GiB, 1967128576 bytes, 3842048 sectors
    Disk model: Storage Device
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0x00000000

    Command (m for help): w
    The partition table has been altered.
    Calling ioctl() to re-read partition table.
    Syncing disks.
    ```

3. Add two partitions:

    - Create a 100MB primary partition of type W95 FAT32 (LBA) 
    - Create another primary partition with the remaining space of type Linux

    ```bash
    $ sudo fdisk /dev/sdb

    Welcome to fdisk (util-linux 2.34).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.


    Command (m for help): n
    Partition type
    p   primary (0 primary, 0 extended, 4 free)
    e   extended (container for logical partitions)
    Select (default p):

    Using default response p.
    Partition number (1-4, default 1):
    First sector (2048-3842047, default 2048):
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-3842047, default 3842047): +100M

    Created a new partition 1 of type 'Linux' and of size 100 MiB.
    Partition #1 contains a vfat signature.

    Do you want to remove the signature? [Y]es/[N]o: Y

    The signature will be removed by a write command.

    Command (m for help): n
    Partition type
    p   primary (1 primary, 0 extended, 3 free)
    e   extended (container for logical partitions)
    Select (default p):

    Using default response p.
    Partition number (2-4, default 2):
    First sector (206848-3842047, default 206848):
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (206848-3842047, default 3842047):

    Created a new partition 2 of type 'Linux' and of size 1.8 GiB.
    Partition #2 contains a ext4 signature.

    Do you want to remove the signature? [Y]es/[N]o: Y

    The signature will be removed by a write command.

    Command (m for help): t
    Partition number (1,2, default 2): 1
    Hex code (type L to list all codes): b

    Changed type of partition 'Linux' to 'W95 FAT32'.

    Command (m for help): p
    Disk /dev/sdb: 1.85 GiB, 1967128576 bytes, 3842048 sectors
    Disk model: Storage Device
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0x00000000

    Device     Boot  Start     End Sectors  Size Id Type
    /dev/sdb1         2048  206847  204800  100M  b W95 FAT32
    /dev/sdb2       206848 3842047 3635200  1.8G 83 Linux

    Command (m for help): w
    The partition table has been altered.
    Calling ioctl() to re-read partition table.
    Syncing disks.
    ```

4. Fromat the partitions:

    ```bash
    # FAT32 for boot partition
    $ sudo mkfs.vfat -F 32 -n boot /dev/sdb1

    # ext4 for root partition
    $ sudo mkfs.ext4 -L root /dev/sdb2
    ```

5. Mount the partitions:
    ```bash
    sudo mount /dev/sdb1 /mnt/boot
    sudo mount /dev/sdb2 /mnt/root
    ```


## Create the Toolchain

The toolchain will consist of:
1. A cross compiler
2. Binary Utilities like the assembler and the linker
3. some runtime libraries

Toolchain is built using toolchain generator called crosstool-ng. 

1. Download crosstool-ng source:
    ```bash
    git clone https://github.com/crosstool-ng/crosstool-ng
    cd crosstool-ng/
    # Switching to 1.24.0 version is necessary. This is because the RPi fork of the Linux Kernel always trails behind the original Linux Kernel, but it is more stable. The kernel version must be higher than the kernel version configured for the toolchain.
    git checkout crosstool-ng-1.24.0 -b 1.24.0
    ```

2. Build and install crosstool-ng

    ```bash
    ./bootstrap
    ./configure --prefix=${PWD} 
    make 
    make install
    export PATH="${PWD}/bin:${PATH}"
    ```

3. Configure crosstool-ng

    By default, crosstool-ng comes with a few ready-to-use-configurations. You can see the full list by typing

    ```bash
    ct-ng list-samples
    ```
    For this version of crosstool-ng, there are no configurations for Rpi4. So an existing configuration for rpi3 is used and is modified to match Rpi4.

    The one from the list that is closest to the target is **aarch64-rpi3-linux-gnu**
    This command is run to get the details of this configuration:
    ```bash
    ct-ng show-aarch64-rpi3-linux-gnu
    ```
    Output:

    ```bash
    [L...]   aarch64-rpi3-linux-gnu
    Languages       : C,C++
    OS              : linux-4.20.8
    Binutils        : binutils-2.32
    Compiler        : gcc-8.3.0
    C library       : glibc-2.29
    Debug tools     : gdb-8.2.1
    Companion libs  : expat-2.2.6 gettext-0.19.8.1 gmp-6.1.2 isl-0.20 libiconv-1.15 mpc-1.1.0 mpfr-4.0.2 ncurses-6.1 zlib-1.2.11
    Companion tools :
    ```

    The OS version is to be noticed. The OS is linux-4.20.8 meaning binaries compiled by the toolchain will run on any kernel version >=4.20.8

    Selecting <b>aarch64-rpi3-linux-gnu</b> as a base-line configuration:

    ```bash
    ct-ng aarch64-rpi3-linux-gnu
    ```

4. Customization for Rpi4:
        
    Open menuconfig:

    ```bash
    ct-ng menuconfig
    ```

    Three changes are to be made:

    - Paths and misc options -> Render the toolchain read-only -> false

    This is for allow extending the toolchain after it is created (by default it is created as read-only)

    - Target Options -> Emit Assembly for CPU -> change <b>cortex-a53</b> to <b>cortex-a72</b>

    - Toolchain Options -> Tuple's Vendor String -> change <b>rpi3</b> to <b>rpi4</b>

5. Building the toolchain

    Before building the toolchain, it is to be made sure that gcc version is 8 and 
    g++ version is 8. Otherwise error will arise while building the toolchain in binutils step.

    If the host system is Ubuntu 20.04 or lower, it is possible to downgrade the gcc and g++ version easily:

    ```bash
    sudo apt-get install -y gcc-8 g++-8
    ```

    Set gcc 8 as default:

    ```bash
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-8 80
    sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-8 80
    ```

    Verify the Installation:

    ```bash
    gcc --version
    g++ --version
    ```

    If the host system is Ubuntu 22.04 (or higher):

    The gcc-8 package has been discontinued in Ubuntu 22.04 and later repositories, but there is a workaround.

    [Workaround for Ubuntu 22.04 or higher](https://askubuntu.com/questions/1446863/trying-to-install-gcc-8-and-g-8-on-ubuntu-22-04)

    Install isl-0.20 and libexpat and other tools:

    ```bash
    wget https://libisl.sourceforge.io/isl-0.20.tar.gz
    mv isl-0.20.tar.gz /home/sprak/crosstool-ng/.build/tarballs/
    ```

    ```bash
    wget https://github.com/libexpat/libexpat/releases/download/R_2_2_6/expat-2.2.6.tar.bz2
    mv expat-2.2.6.tar.bz2 /home/surya/crosstool-ng/.build/tarballs/
    ```

    ```bash
    sudo apt-get install -y python3-dev
    ```

    Finally, build the configured crosstool-ng:

        ```bash
        ct-ng build
        ```

## Build the Bootloader

Because bootloader is device specific, it is to be configured before building like crosstool-ng.

Here, U-boot is used for the proprietory ROM code.

1. Clone U-Boot repository:

    ```bash
    git clone git://git.denx.de/u-boot.git 
    cd u-boot
    git checkout v2021.10 -b v2021.10
    ```

2. Configure U-Boot:

    ```bash
    export PATH=${HOME}/x-tools/aarch64-rpi4-linux-gnu/bin/:$PATH
    export CROSS_COMPILE=aarch64-rpi4-linux-gnu-
    make rpi_4_defconfig
    ```

3. Build U-boot:

    ```bash
    make
    ```

4. Copy U-boot binaries onto SD Card:

    ```bash
    sudo cp u-boot.bin /mnt/boot
    ```

Raspberry Pi has its own bootloader, which is loaded by the ROM code and is capable of loading the kernel. However, since the open source u-boot is being used, Raspberry Pi bootloader has to be configured to load u-boot and then let u-boot load the kernel.

```bash
# Download Raspberry Pi firmware/boot directory
svn checkout https://github.com/raspberrypi/firmware/trunk/boot

# Copy Raspberry Pi 4 bootloader into boot partition
sudo cp boot/{bootcode.bin,start4.elf} /mnt/boot/

# Let Raspberry Pi 4 bootloader load u-boot
cat << EOF > config.txt
enable_uart=1
arm_64bit=1
kernel=u-boot.bin
EOF
$ sudo mv config.txt /mnt/boot/
```

## Build the Kernel

1. Clone the kernel repository:
    ```bash
    git clone --depth=1 -b rpi-5.10.y https://github.com/raspberrypi/linux.git
    cd linux
    ```

2. Configure and build the kernel:
    ```bash
    make ARCH=arm64 CROSS_COMPILE=aarch64-rpi4-linux-gnu- bcm2711_defconfig
    make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-rpi4-linux-gnu-
    ```

3. Copy the kernel and device tree onto the SD Card:

    ```bash
    sudo cp arch/arm64/boot/Image /mnt/boot
    sudo cp arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb /mnt/boot/
    ```

## Build the Root Filesystem

1. Create Directories:

    ```bash
    mkdir rootfs
    cd rootfs
    mkdir {bin,dev,etc,home,lib64,proc,sbin,sys,tmp,usr,var}
    mkdir usr/{bin,lib,sbin}
    mkdir var/log

    # Create a symbolink lib pointing to lib64
    ln -s lib64 lib

    tree -d
    .
    ├── bin
    ├── dev
    ├── etc
    ├── home
    ├── lib -> lib64
    ├── lib64
    ├── proc
    ├── sbin
    ├── sys
    ├── tmp
    ├── usr
    │   ├── bin
    │   ├── lib
    │   └── sbin
    └── var
        └── log

    16 directories

    # Change the owner of the directories to be root
    # Because current user doesn't exist on target device

    sudo chown -R root:root *
    ```

2. Installing Shell and other Linux Utilities:

    Here Busybox is used.

    ```bash
    # Download the source code
    wget https://busybox.net/downloads/busybox-1.33.2.tar.bz2
    tar xf busybox-1.33.2.tar.bz2
    cd busybox-1.33.2/

    Config
    CROSS_COMPILE=${HOME}/x-tools/aarch64-rpi4-linux-gnu/bin/aarch64-rpi4-linux-gnu-
    make CROSS_COMPILE="$CROSS_COMPILE" defconfig
    # Change the install directory to be the one just created
    sed -i 's%^CONFIG_PREFIX=.*$%CONFIG_PREFIX="/home/hechaol/rootfs"%' .config

    # Build
    make CROSS_COMPILE="$CROSS_COMPILE"

    # Install
    # Use sudo because the directory is now owned by root
    sudo make CROSS_COMPILE="$CROSS_COMPILE" install
    ```

3. Install required libraries:

    ```bash
    readelf -a ~/rootfs/bin/busybox | grep -E "(program interpreter)|(Shared library)"
      [Requesting program interpreter: /lib/ld-linux-aarch64.so.1]
    0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]
    0x0000000000000001 (NEEDED)             Shared library: [libresolv.so.2]
    0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
    ```

    Copy these files from the toolchain's sysroot directory to the rootfs/lib directory.

    ```bash
    export SYSROOT=$(aarch64-rpi4-linux-gnu-gcc -print-sysroot)
    sudo cp -L ${SYSROOT}/lib64/{ld-linux-aarch64.so.1,libm.so.6,libresolv.so.2,libc.so.6} ~/rootfs/lib64/
    ```

    Create Device Nodes:

    ```bash
    cd ~/rootfs
    sudo mknod -m 666 dev/null c 1 3
    sudo mknod -m 600 dev/console c 5 1
    ```


## Preparing the Root Partition

Insert the SD card into the card reader and the reader into the desktop.

1. Copying rootfs onto the Root Partition of the SD Card:

```bash
sudo mount /dev/sdb1 /mnt/root
sudo mount /dev/sdb2 /mnt/root
sudo cp -r ~/rootfs/* /mnt/root/
```

2. Change boot commands

```bash
cat << EOF > boot_cmd.txt
fatload mmc 0:1 \${kernel_addr_r} Image
setenv bootargs "console=serial0,115200 console=tty1 root=/dev/mmcblk0p2 rw rootwait init=/bin/sh"
booti \${kernel_addr_r} - \${fdt_addr}
EOF
~/u-boot/tools/mkimage -A arm64 -O linux -T script -C none -d boot_cmd.txt boot.scr
sudo cp boot.scr /mnt/boot/
```

## Boot

Unmount the partitions and insert the SD Card into the Raspberry Pi 4:

```bash
sudo umount /dev/sdb1
sudo umount /dev/sdb2
```

After powering up the Raspberry Pi 4, the Busybox shell shows up if successfully booted.
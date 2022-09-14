
# *Dropbear* SSH server on *Raspberry Pi 4*

## Hardware requirements
- Host machine (*Ubuntu 22.04*)
- *Raspberry Pi 4*
- USB-to-Serial converter
- Network cable
- SD card

## Steps to follow 
- Host configuration
- Building the compiler (by *Crosstool-NG*)
- Building the custom kernel
- Building *BusyBox* (to make *rootfs* and *init*)
- Building *zlib* (to make the shared library)
- Bulding *Dropbear* (to make the SSH server)
- Building *U-Boot* (to make the bootloader)
- Transferring artifacts

## Host configuration
We use *Docker* containers as much as possible to make the minimum side effects on the host. You can get the [*Dockerfiles*](https://github.com/yasin-gorgij/dockerfiles) and build the required images (*busybox*,  *crosstool-ng*, *Dropbear*, *rpi-kernel*, *u-boot*, and *zlib*) or you can install them on your host machine if you prefer.
To transfer artifacts to *Raspberry Pi 4* we use two mediums:
- Network (by using tftp and NFS)
- SD card 

On the host we need some applications:
- NFS server (to make root filesystem accessible over the network)
- picocom (to get access to *Raspberry Pi 4* over serial port)
- tftp server (to make *Raspberry Pi* custom kernel accessible over the network)
- vim (to edit files)

### Installation
sudo apt-get install -y nfs-kernel-server picocom tftpd-hpa vim
To install *Docker* follow [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu) instructions.

### Configuration
*NFS* server configuration to share the root filesystem with the *Raspberry Pi*:
- Edit *NFS* server configuration file `sudo vim /etc/exports`
- Add `/tmp/nfs 192.168.2.0/24(rw,sync,no_root_squash,no_subtree_check)` to the end of the file
- Create */tmp/nfs* directory `mkdir -p /tmp/nfs`
- Apply the changes to *NFS* server `sudo exportfs -ar`
- Check the *NFS* server status `sudo systemctl status nfs-kernel-server.service`

*tftp* server configurationto share the custom kernel with the *Raspberry Pi*:
- Edit *tftp* server configuration file `sudo vim /etc/default/tftpd-hpa`
- Change the value of *TFTP_DIRECTORY* to `/tmp/tftp`
- Create */tmp/tftp* directory `mkdir -p /tmp/tftp`
- Check *tftp* server status `sudo systemctl status tftpd-hpa.service`

### Required artifacts
In order to boot *Raspberry Pi 4* you have to download `start4.elf`:
- `wget --directory-prefix=$HOME/dev/artifacts/toolchain/rpi4/bootloader https://raw.githubusercontent.com/raspberrypi/firmware/master/boot/start4.elf`

Because we want to boot it with our own custom kernel we have to change the boot process by creating a configuration file:
- Create *config.txt* file `vim $HOME/dev/artifacts/toolchain/rpi4/bootloader/config.txt`
- Add the following lines to it:
  ```
  enable_uart=1
  kernel=u-boot.bin
  arm_64bit=1
  ```

## Building the compiler
*Crosstool-NG* builds a very basic tool for us which is a compiler that required for every other steps.

Run the *Crosstool-NG* *Docker* container:
`docker container run --rm -it -v $HOME/dev/artifacts/toolchain:/toolchain crosstool-ng:1.25.0`

In the container you can view the list of sample configuration files in *Crosstool-NG* by `ct-ng list-samples` command.

Configure *Crosstool-NG*:
- Create the configuration `ct-ng aarch64-rpi4-linux-gnu`
- Edit  the configuration `ct-ng menuconfig`
  * This action opens a menu for us to make the changes we want:
    * Enter to **Path and misc options** menu and change **Prefix directory** to `${CT_PREFIX:-/toolchain/x-tools}/${CT_HOST:+HOST-${CT_HOST}/}${CT_TARGET}` and **Exit** to main menu
    * Enter **Operating System** menu and change **Version of linux** to `5.15.23` and **Exit** to main menu
    * Enter **C-library** menu and change **Minimum supported kernel version** to `Let ./configure decide` and **Exit** to main menu

Now **Save** changes and **Exit** then run `ct-ng build`.

After finishing the build process successfully you'll have the artifacts in `$HOME/dev/artifacts/toolchain/x-tools` directory of your host machine.

Now you can exit from the container.

## Building *U-Boot*
*U-Boot* creates an intermediate kernel which we call it *bootloader*. it's job is to load the final kernel if we wanted to, which we want here in this project. It also gives us some limited functionalities to work with our target hardware (here, *Raspberry Pi 4*).

Run the *U-Boot* *Docker* container:
`docker container run --rm -it -v $HOME/dev/artifacts/toolchain:/toolchain u-boot:v2022.04`

Configure and build *U-Boot*:
- `export CROSS_COMPILE=aarch64-rpi4-linux-gnu-`
- `export PATH=$PATH:/toolchain/x-tools/aarch64-rpi4-linux-gnu/bin`
- Create the configuration `make rpi_4_defconfig`
- View other configurations `ls configs` (optional step)
- Make *U-Boot* kernel image (bootloader) `make -j$(nproc)`

After finishing the build process copy *u-boot.bin* to *bootloader* directory:
- `cp /u-boot-v2022.04/u-boot.bin /toolchain/rpi4/bootloader/`

Now you can exit from the container.

## Building the custom kernel
In this step we'll build our own custom kernel for *Raspberry Pi 4* by the compiler we built in *Building the compiler* step.

Run the *Raspberry Pi* kernel *Docker* container:
`docker container run --rm -it -v $HOME/dev/artifacts/toolchain:/toolchain rpi-kernel:all-versions`

Configure the kernel:
- `export ARCH=arm64`
- `export CROSS_COMPILE=aarch64-rpi4-linux-gnu-`
- `export PATH=$PATH:/toolchain/x-tools/aarch64-rpi4-linux-gnu/bin`
- Change the branch to *rpi-5.15.y* `git switch rpi-5.15.y`
- Create the kernel configuration for *Raspberry Pi 4* `make bcm2711_defconfig` command

Now run `make -j$(nproc)`.
Congratulation! now you have your custom kernel image.
Copy the kernel and the device tree:
- `cp /linux/arch/arm64/boot/Image /toolchain/rpi4/bootloader/`
- `cp /linux/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb /toolchain/rpi4/bootloader/`

You can also build the kernel modules by `INSTALL_MOD_PATH=/toolchain/rpi4/rootfs make modules_install` but we don't need them in this project.

Now you can exit from the container.

## Building *BusyBox*
In this step we'll build a root filesystem (*rootfs*) and an initializing program (*init*) for the custom kernel we built in *Building the custom kernel* step.

Run the *BusyBox* *Docker* container:
`docker container run --rm -it -v $HOME/dev/artifacts/toolchain:/toolchain busybox:1.34.1`

Configure *BusyBox*:
- `export CROSS_COMPILE=aarch64-rpi4-linux-gnu-`
- `export PATH=$PATH:/toolchain/x-tools/aarch64-rpi4-linux-gnu/bin`
- Create the configuration `make defconfig`
- Edit the configuration `make menuconfig`:
  * Go to **Settings** and in **Installation Options ("make install" behavior)** category and change **Destination path for 'make install'** option to `/toolchain/rpi4/rootfs`

Now **Exit** to main menu and save the changes then run:
- `make -j$(nproc)`
- `make install`

Since we built *BusyBox* dynamically we have to find out which libraries it depends on and copy them:
- Find out dependencies ``aarch64-rpi4-linux-gnu-ldd --root `aarch64-rpi4-linux-gnu-gcc --print-sysroot` /toolchain/rpi4/rootfs/bin/busybox``
- Copy dependencies:
  * `mkdir -p /toolchain/rpi4/rootfs/lib`
  * `cp /toolchain/x-tools/aarch64-rpi4-linux-gnu/aarch64-rpi4-linux-gnu/sysroot/lib/ld-linux-aarch64.so.1 /toolchain/rpi4/rootfs/lib`
  * `cp /toolchain/x-tools/aarch64-rpi4-linux-gnu/aarch64-rpi4-linux-gnu/sysroot/lib/libresolv.so.2 /toolchain/rpi4/rootfs/lib`
  * `cp /toolchain/x-tools/aarch64-rpi4-linux-gnu/aarch64-rpi4-linux-gnu/sysroot/lib/libc.so.6 /toolchain/rpi4/rootfs/lib`
  * `cp /toolchain/x-tools/aarch64-rpi4-linux-gnu/aarch64-rpi4-linux-gnu/sysroot/lib/libm.so.6 /toolchain/rpi4/rootfs/lib`
  * `ln -s lib /toolchain/rpi4/rootfs/lib64`
  * ``cp `aarch64-rpi4-linux-gnu-gcc --print-sysroot`/sbin/ldconfig /toolchain/rpi4/rootfs/sbin/``

we also have to create the following files and directories that will be used by the kernel:
```
mkdir -p /toolchain/rpi4/rootfs/dev /toolchain/rpi4/rootfs/etc/dropbear /toolchain/rpi4/rootfs/etc/init.d /toolchain/rpi4/rootfs/proc /toolchain/rpi4/rootfs/sys /toolchain/rpi4/rootfs/var/log /toolchain/rpi4/rootfs/var/run
```
```
touch /toolchain/rpi4/rootfs/etc/group /toolchain/rpi4/rootfs/etc/passwd /toolchain/rpi4/rootfs/etc/shadow
```

*BusyBox* can run a boot time shell script called *rcS* and we use it to mount *proc* and *sysfs* and start the *SSH* server in background.
Create and edit *rcS* shell script by `vim /toolchain/rpi4/rootfs/etc/init.d/rcS` and add the following contents to it:
```
#!/bin/sh

mount -t proc nodev /proc
mount -t sysfs nodev /sys

mkdir -p /dev/pts
mount -t devpts nodev /dev/pts

ldconfig
dropbear -ER
```
And don't forget to make it executable: `chmod +x /toolchain/rpi4/rootfs/etc/init.d/rcS`

Now you can exit from the container.

## Building *zlib*
In this step we'll build *zlib* which is *Dropbear* depends on it.

Run the *zlib* *Docker* container:
`docker container run --rm -it -v $HOME/dev/artifacts/toolchain:/toolchain zlib:1.2.12`

Now build it:
- `export CROSS_PREFIX=aarch64-rpi4-linux-gnu-`
- `export PATH=$PATH:/toolchain/x-tools/aarch64-rpi4-linux-gnu/bin`
- `./configure --prefix=/usr`
- `make install DESTDIR=/toolchain/rpi4/rootfs`

You can exit from the container.

## Bulding *Dropbear*
In this step we'll build *Dropbear* itself.

Run the *Dropbear* *Docker* container:
`docker container run --rm -it -v $HOME/dev/artifacts/toolchain:/toolchain dropbear:2022.82`

Now build it:
- `export PATH=$PATH:/toolchain/x-tools/aarch64-rpi4-linux-gnu/bin`
- `./configure --host=aarch64-rpi4-linux-gnu --prefix=/usr --with-zlib=/toolchain/rpi4/rootfs/usr`
- `make -j$(nproc)`
- `make install DESTDIR=/toolchain/rpi4/rootfs`
-  Find out dependencies ``aarch64-rpi4-linux-gnu-ldd --root `aarch64-rpi4-linux-gnu-gcc --print-sysroot` /toolchain/rpi4/rootfs/usr/sbin/dropbear``
- Copy dependencies:
  * `cp /toolchain/x-tools/aarch64-rpi4-linux-gnu/aarch64-rpi4-linux-gnu/sysroot/lib/libcrypt.so.1 /toolchain/rpi4/rootfs/lib`
- Create files for login/logout logs:
  ```
  touch /toolchain/rpi4/rootfs/var/log/lastlog /toolchain/rpi4/rootfs/var/log/btmp /toolchain/rpi4/rootfs/var/log/wtmp /toolchain/rpi4/rootfs/var/run/utmp
  ```

You can exit from the container.

## Transferring artifacts
It's time to transfer the artifacts, we use two mediums to do that:
- *NFS*: which is a perfect choice when you're developing
- SD card: when you finished the development and want to transfer final artifacts to the target hardware

### *NFS* and *tftp*
It has to be said that even in *NFS* transferring method we need to copy some files to the SD card but it's a one time process.

*NFS* server configuration is done in **Host configuration** step and now it's time to configure the client (*Raspberry Pi 4*). Client configuration can be done after booting *Raspberry Pi 4* with *U-Boot* kernel so before booting it we have to create a boot partition on the SD card.

Put the SD card in a SD card reader and plug it to USB port of the host machine or if the host machine has a built-in SD card reader, put it in that slot.
> Warning: In this all of the data on the SD card will be removed. If you have any data, back them up

In Ubuntu the SD card will be mounted on `/media` you can be sure by running `lsblk | grep media` and see the result, the right-hand side of the result is the device which for me is `sda` and the left-hand side is mount point which starts with  `/media/...`.

Partitioning and formatting:
- Unmount the SD card `sudo umount /media/$USER/*`
- Start partitioning `sudo cfdisk /dev/sda`
  * In *cfdisk* **Delete** all partitions and add a **New** `50M` (50 MB) **primary** partition.
  * Change the **Type** to **c W95 FAT32 (LBA)** and **Write** the changes by typing **yes**, then **Quit** from *cfdisk*.
- Format the partition `sudo mkfs.vfat -F 32 /dev/sda1`

Copy the required data to the SD card:
- Create the mount point `mkdir -p /tmp/rpi4-bootloader`
- Mount the SD card `sudo mount -o rw,uid=$USER,gid=$USER -t vfat /dev/sda1 /tmp/rpi4-bootloader`
- Copy the data:
  * `cp ~/dev/artifacts/toolchain/rpi4/bootloader/bcm2711-rpi-4-b.dtb /tmp/rpi4-bootloader/`
  * `cp ~/dev/artifacts/toolchain/rpi4/bootloader/config.txt /tmp/rpi4-bootloader/`
  * `cp ~/dev/artifacts/toolchain/rpi4/bootloader/start4.elf /tmp/rpi4-bootloader/`
  * `cp ~/dev/artifacts/toolchain/rpi4/bootloader/u-boot.bin /tmp/rpi4-bootloader/`

Remove the SD card from the host machine safely by running `sudo umount /tmp/rpi4-bootloader` and place it in the *Raspberry Pi* SD card slot. Connect network cable, and *USB-to-Serial* converter to the *Raspberry Pi* and the other end of each cable to the host machine.

Share the data on *NFS*/*tftp* server side:
- Copy the custom kernel to *tftp* server directory `cp ~/dev/artifacts/toolchain/rpi4/bootloader/Image /tmp/tftp/`
- Copy the root filesystem to *NFS* server directory `cp -r ~/dev/artifacts/toolchain/rpi4/rootfs/* /tmp/nfs`

Start *picocom* to interact with the *Raspberry Pi* through serial port: `sudo picocom -b 115200 /dev/ttyUSB0`. Now power on the *Raspberry pi* and just wait for **Hit any key to stop autoboot** message to be appear and immediately press a key. You have to see this `U-Boot>`.

Now it's time to configure the *NFS* client:
- Setting the load path of our custom kernel
- Setting the root filesystem path from *NFS* server

To set the load path of our custom kernel we configure the *tftp* client in *U-Boot*:
- `setenv serverip 192.168.2.100`, serverip is our host machine IP
- `setenv netmask 255.255.255.0`
- `setenv ipaddr 192.168.2.110`, ipaddr is our *Raspberry Pi 4* IP address
- `setenv bootcmd 'tftp ${kernel_addr_r} Image; fatload mmc 0:1 ${fdt_addr_r} bcm2711-rpi-4-b.dtb; booti ${kernel_addr_r} - ${fdt_addr_r}'`
- `saveenv`

To set the root filesystem from *NFS* server:
- `setenv bootargs console=ttyS0,115200 8250.nr_uarts=1 swiotlb=512 root=/dev/nfs ip=192.168.2.110 nfsroot=192.168.2.100:/tmp/nfs,nfsvers=4.2,tcp init=/linuxrc rw`
- `saveenv`

Because it takes time for network connection to be ready we can increase the boot delay time to 10 seconds:
- `setenv bootdelay 10`
- `saveenv`
- `reset`

After resetting wait for the kernel and the root filesystem to be loaded from network, it takes time so be patient. When you see **Please press Enter to activate this console.** message everything is ready then press *Enter*.
To be able to *SSH* to the *Raspberry Pi* you have to create a password for *root* user:
- `addgroup -g 0 root`
- `adduser -h /root -G root -u 0 root` and set password

Congratulations! Now you have your own *SSH* server on the *Raspberry Pi* that you built.
You can connect to it from the host machine: `ssh root@192.168.2.110`

> Note: You can copy the environment variable file (*uboot.env*) from the SD card for later use.

### *SD card*
When the development is finished you need to prepare data to copied/written to SD card.
We create some directories to place the different data that we need into them:
- Bootloader directory: `mkdir -p /tmp/rpi4-bootloader`
- Root filesystem directory: `mkdir -p /tmp/rpi4-rootfs`

Data of each directory:
- Bootloader data
  * `cp ~/dev/artifacts/toolchain/rpi4/bootloader/* /tmp/rpi4-bootloader/`
- Other directories data
  * `cp -r ~/dev/artifacts/toolchain/rpi4/rootfs/* /tmp/rpi4-rootfs/`
  * Edit *rcS* script by `vim /tmp/rpi4-rootfs/etc/init.d/rcS` and change the contents to:
    ```
    #!/bin/sh

    mount -t proc nodev /proc
    mount -t sysfs nodev /sys

    mkdir -p /dev/pts
    mount -t devpts nodev /dev/pts

    ifconfig eth0 192.168.2.110 up

    ldconfig
    dropbear -ER
    ```
    
  * Make the script executable by `chmod +x /tmp/rpi4-rootfs/etc/init.d/rcS`
  * We can strip binary and library files to have smaller version of them:
    * `~/dev/artifacts/toolchain/x-tools/aarch64-rpi4-linux-gnu/bin/aarch64-rpi4-linux-gnu-strip /tmp/rpi4-rootfs/bin/*` 
    * `~/dev/artifacts/toolchain/x-tools/aarch64-rpi4-linux-gnu/bin/aarch64-rpi4-linux-gnu-strip /tmp/rpi4-rootfs/usr/bin/*`
    * `~/dev/artifacts/toolchain/x-tools/aarch64-rpi4-linux-gnu/bin/aarch64-rpi4-linux-gnu-strip /tmp/rpi4-rootfs/usr/lib/*`
    * `~/dev/artifacts/toolchain/x-tools/aarch64-rpi4-linux-gnu/bin/aarch64-rpi4-linux-gnu-strip /tmp/rpi4-rootfs/usr/sbin/*`

We create and format partitions in this step. We'll create the following partitions:
- A 50 MB bootloader partition of type *c W95 FAT32 (LBA)*
- A partition of type Ext4 and its size will be the remaining free space of the SD card for root filesystem

Put the SD card in a SD card reader and plug it to USB port of the host machine or if the host machine has a built-in SD card reader, put it in that slot.
> Warning: In this all of the data on the SD card will be removed. If you have any data, back them up

Before partitioning make sure to unmount the SD card `sudo umount /media/$USER/*`
Start partitioning by `sudo cfdisk /dev/sda` (*sda* is my SD card) and **Delete** all partitions.
- Add a **New** `50M` (50 MB) **primary** partition of **Type** to **c W95 FAT32 (LBA)** for *bootloader*
- Add a **New** (the rest of free space) **primary** partition for root filesystem

Then **Write** the changes by typing **yes**, then **Quit** from *cfdisk*.
It's time to format partitions
- Format bootloader partition: `sudo mkfs.vfat -F 32 /dev/sda1`
- Format *etc* partition: `sudo mkfs.ext4 /dev/sda2`

Now we copy/write the data to SD card
* Copy bootloader data
  * Create the mount point `mkdir -p /tmp/rpi4-bootloader-partition`
  * Mount the partition `sudo mount -o rw,uid=$USER,gid=$USER -t vfat /dev/sda1 /tmp/rpi4-bootloader-partition`
  * Copy data `cp /tmp/rpi4-bootloader/* /tmp/rpi4-bootloader-partition/`
* Copy data to root filesystem partition
  * Create the mount point `mkdir -p /tmp/rootfs-partition`
  * Mount the partition `sudo mount -t ext4 /dev/sda2 /tmp/rootfs-partition`
  * Give read/write access to the current user `sudo chown $USER:$USER /tmp/rootfs-partition`
  * Remove unnecessary files and directories `rm -rf /tmp/rootfs-partition/*`
  * Copy data `cp -r /tmp/rpi4-rootfs/* /tmp/rootfs-partition`

Don't forget to unmout the partitions before putting the SD card in the *Raspberry Pi* SD card slot.
- `sudo umount /tmp/rpi4-bootloader-partition`
- `sudo umount /tmp/rootfs-partition`

There is one more step and the step is defining environment variables for *U-Boot* so start *picocom* on host machine `sudo picocom -b 115200 /dev/ttyUSB0` and power on the *Raspberry Pi* then wait for **Hit any key to stop autoboot** message to be appear and immediately press a key. You have to see this `U-Boot>` and then you can set the environment variables:
- `setenv bootargs console=ttyS0,115200 8250.nr_uarts=1 swiotlb=512 root=/dev/mmcblk0p2 rootwait rootfstype=ext4 rw init=/linuxrc`
- `setenv bootcmd 'fatload mmc 0:1 ${kernel_addr_r} Image; fatload mmc 0:1 ${fdt_addr_r} bcm2711-rpi-4-b.dtb; booti ${kernel_addr_r} - ${fdt_addr_r}'`
- `saveenv`
- `reset`

After resetting wait for the kernel and the root filesystem to be loaded. When you see **Please press Enter to activate this console.** message everything is ready then press *Enter*.


To be able to *SSH* to the *Raspberry Pi* you have to create a password for *root* user:
- `addgroup -g 0 root`
- `adduser -h /root -G root -u 0 root` and set password

Congratulations! Now you have your own *SSH* server on the *Raspberry Pi* that you built.
You can connect to it from the host machine: `ssh root@192.168.2.110`

> Note: You can copy the environment variable file (*uboot.env*) from the SD card for later use.

# License
The repository contents are released under [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)



# **DRAKVUF on ARM**

While DRAKVUF is an effective binary analysis system, it's services were limited exclusively to x86 (and x86-64) systems, until now! This tutorial is a comprehensive step-by-step guide to setting up DRAKVUF on ARM. All steps can be performed by using a Linux (in our case a Debian Jessie) system. Within the scope of this tutorial, we used a [Hikey Lemaker](https://www.96boards.org/product/hikey/ "96boards Homepage") development board implementing an ARM Cortex-A53 Octa-Core and 2GB of RAM. While the following steps can be applied to arbitrary development boards, the user must bear in mind that the selected board requires [Xen support](https://wiki.xenproject.org/wiki/Category:XenARM "Xen support for ARM").

From the bird's eye perspective, in the following, we will show how to:
  * Set up a Linux from scratch and Xen including Dom0 and DomU on ARM.
    * We will set up an SD-Card and
    * debootstrap a minimalistic filesystem
  * Cross-compile Linux and Xen on our host to speed-up compilation for the ARM board
  * Compile and install LibVMI and DRAKVUF on ARM
  * Use DRAKVUF on ARM


## **Preparing the SD-card**

First of all, in order to provide a suitable environment for Xen including Dom0 and DomU domains, we have to partition an SD-card that comes with the board. In total, we will proivde three physical partitions. First, as the Hikey Lemaker development board implements an UEFI-based system, we need an `EFI partition` for holding EFI-images, which should be at least `100 MB` of size. Second, to host the *Dom0* and *DomU* domains, we need two further domains. While we create a `7GB` partition for both Dom0 and DomU, the sizes can be adjusted according to your needs.

Usually, mount points of SD-cards have the following format: `/dev/sdx` (whereas x is a random letter that depends, e.g., on how many harddrives are currently connected to your system). Yet, sometimes the names differ. To find out which mount point is associated with the SD-card, the following command can be of great help:

```bash
sudo lsblk
```

With this information, the SD-card can set up in the following way by using the good old `fdisk` (in the following, we will use `/dev/sdx` to symbolize the mount point associated with the SD-card):

  * EFI partition:
  ```bash
  sudo fdisk /dev/sdx
  > g       # create a new GPT
  > n       # create a new partition
  > 1       # assign it a partition number, here: 1
  >         # confirm the start of the partition
  > +100M   # allocate 100 MB for the EFI partition
  > t       # change the type of the partition
  > L       # list all available partition types
  > 1       # select the type 'EFI System'
  ```

  * Dom0 partition:
  ```bash
  > n       # create a new partition
  > 2       # assign it a partition number, here: 2
  >         # confirm the start of the partition
  > +7G     # allocate 7 GB for the Dom0 partition
  ```

  * DomU partition:
  ```bash
  > n       # create a new partition
  > 3       # assign it a partition number, here: 3
  >         # confirm the start of the partition
  > +7G     # allocate 7 GB for the Dom0 partition
  > w       # write table to SD card and exit
  ```

To verify that our instructions have been successfully executed, we can invoke the following command to print the created partition table:

```bash
sudo fdisk -l /dev/sdx
```

After having prepared the upper partitions, we will need to format them with an appropriate filesystem. We choose to use `fat32` for the `EFI partition` and `ext4` for the both `Dom0` and `DomU partitions`:

```bash
sudo mkfs.fat -F32 /dev/sdx1
sudo mkfs.ext4 -L dom0 /dev/sdx2
sudo mkfs.ext4 -L domu /dev/sdx3
```

## **Debootstrapping a Debian File System**

As soon as the partitions have been created and formatted accordingly, we can create a Debian file system for ARM by using `(qemu-)debootstrap`. The tool `qemu-debootstrap` is a wrapper for the original `debootstrap` and simplifies the process of bootstrapping filesystems for different architectures, which makes it especially handy for our purposes. Therefore, we first need to install the associated libraries:

```bash
sudo apt install qemu qemu-user-static debootstrap binfmt-support
```

Now, we are ready to `debootstrap` the file system on both the `Dom0` and `DomU patition`:

```bash
mkdir /mnt/{dom0,domu,efi}

export DOM0=/mnt/dom0
export DOMU=/mnt/domu

sudo mount /dev/sdx2 $DOM0
sudo mount /dev/sdx3 $DOMU

sudo qemu-debootstrap --arch arm64 stretch $DOM0 <preferred-debian-mirror>
sudo qemu-debootstrap --arch arm64 stretch $DOMU <preferred-debian-mirror>

sudo umount $DOM0
sudo umount $DOMU
```



## **Configuring Dom{0|U}**

Before cross-compiling the Linux kernel and Xen, we should provide a basic user environment for (at least) Dom0.

  ---
  ### **NOTE:** `debootstrap`

  In case you have used plain `debootstrap` above (instead of `qemu-debootstrap`), you should copy the `qemu-arm-static` into the Dom0 filesystem and configure the  `binfmt` subsystem to be able to interpret ELF binaries compiled for AArch32 (if you have not already done so).

  ```bash
    sudo mount /dev/sdx2 $DOM0

    cp /usr/bin/qemu-aarch64-static $DOM0/usr/bin/.
    echo ':qemu-aarch64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-aarch64-static:OCF' > /proc/sys/fs/binfmt_misc/register

    sudo umount $DOM0
  ```

  ---

Now, `chroot` into the previously prepared Dom0 partition:

```bash
sudo chroot $DOM0           # on Debian, chroot is able to switch into the ARM partitions
```

---
  ### **NOTE:** Non-Debian Systems

  In case you should work on a non-Debian Linux, you can use the following command to set `chroot` into the Dom0 partition. In case you cannot execute any commands (e.g., `ls`), please make sure that the required paths in `$PATH` environment variable are set correctly.

  ```bash
    sudo chroot $DOM0 qemu-arm-static /bin/bash
  ```
---

At this point, it is up to you what you would like to prepare for your initital Dom0 filesystem. For instance, you can set up user accounts and change passwords through `passwd`, set up networking in `/etc/network/interfaces`, and mount required partitions through `/etc/fstab`:

```bash
vim /etc/fstab

/dev/mmcblk1p1      /boot vfat   defaults          0 2
/dev/mmcblk1p2      /     ext4   defaults          0 1
```

But more importantly, you can install packages that you will need for further operation:

```bash
apt-get update
apt install vim vim-syntax-gtk build-essential make gcc
apt install libyajl-dev libfdt-dev libaio-dev             # Needed by Xen-tools
apt install openssh-server openssh-client
```



## **Cross-Compiling the Linux Kernel**

Cross-compiling the Linux kernel simplifies the overall development process. Instead of compiling the kernel on a development board with potentially only limited resources, we can use our x86-based development system. This step becomes especially relevant if the Linux kernel has to be recompiled often during the development process. As such, in the following, we show how to cross-compile the Linux kernel for the AArch64 architecture.

Before we can cross-compile a Linux kernel, we need to make sure that our host system has all packages installed that are needed by further operation (the following list might be incomplete):

```bash
sudo apt install build-essential make gcc gcc-arm-linux-gnueabi gcc-aarch64-linux-gnu ncurses-dev bison flex libssl-dev git
```

Now, get the most recent (or rather the one with sufficient support for your ARM development board) Linux kernel and cross-compile it for AArch64:

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.xx.x.tar.xz
tar xaf linux-4.xx.x.tar.xz && cd linux-4.xx.x

export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

make defconfig
make menuconfig     # configure your kernel according to your needs
```

To enable Xen support, make sure that your Linux kernel configuration enables the following configuration parameters:

```bash
CONFIG_XEN_DOM0
CONFIG_XEN
CONFIG_XEN_BLKDEV_FRONTEND
CONFIG_XEN_BLKDEV_BACKEND
CONFIG_XEN_NETDEV_FRONTEND
CONFIG_INPUT_XEN_KBDDEV_FRONTEND
CONFIG_HVC_XEN
CONFIG_HVC_XEN_FRONTEND
CONFIG_XEN_FBDEV_FRONTEND
CONFIG_XEN_BALLOON
CONFIG_XEN_SCRUB_PAGES
CONFIG_XEN_DEV_EVTCHN
CONFIG_XEN_BACKEND
CONFIG_XENFS
CONFIG_XEN_COMPAT_XENFS
CONFIG_XEN_SYS_HYPERVISOR
CONFIG_XEN_XENBUS_FRONTEND
CONFIG_XEN_GNTDEV
CONFIG_XEN_GRANT_DEV_ALLOC
CONFIG_SWIOTLB_XEN
CONFIG_XEN_PRIVCMD
CONFIG_XEN_EFI
CONFIG_XEN_AUTO_XLATE
```

Finally cross-compile your Linux kernel and the DTB (note that the DTB depends on your board). To reduce compilation time, please adjust the number of cores that shall be used for the process of compilation. This number is provided by the `-j` switch (in our case, we use 7 cores):

```bash
make -j7 Image modules hisilicon/hi6220-hikey.dtb
```

After the kernel has been successfully compiled, you should install the compiled modules and the kernel itself to the `EFI partition`. Please note that we have to export the environment variables as switching to the `root` user changes the environment and with it all previously exported environment variables:


```bash
sudo su

export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
export EFI=/mnt/efi
export DOM0=/mnt/dom0

mount /dev/sdx1 $EFI
mount /dev/sdx2 $DOM0

make INSTALL_MOD_PATH=$DOM0 -j7 modules_install
cp ./arch/arm64/boot/{Image,dts/hisilicon/hi6220-hikey.dtb} $EFI

umount $EFI
umount $DOM0
sync

exit
```


## **Cross-Compiling Xen**

Now that we have set up our domains, we should cross-compile and install the
Xen hypervisor. As DRAKVUF requires the [Xen altp2m subsystem](http://arm-altp2m.blogspot.com/ "Xen altp2m on ARM") to work properly on ARM, we suggest to clone this modified Xen v4.11 [repository](https://github.com/sergej-proskurin/xen/tree/arm-altp2m-drakvuf "Xen altp2m on ARM for DRAKVUF"). Make sure that you use the branch `arm-altp2m-drakvuf`.

```bash
export CROSS_COMPILE=aarch64-linux-gnu-

make -j7 dist-xen XEN_TARGET_ARCH=arm64
```

Now copy the compiled Xen EFI-image to the `EFI partition`.


```bash
sudo su

export EFI=/mnt/efi

mount /dev/sdx1 $EFI
cp ./xen/xen.efi $EFI/xen.efi
```

To make Xen to use the previously compiled Linux kernel and DTB for Dom0, we need
to create the following configuration files on the `EFI partition` (note that
this configuration is highly hardware dependent and might vary on your board):


 * Config File: `xen.cfg`:

 ```bash
 vim $EFI/xen.cfg

 options=console=dtuart dom0_mem=1024M,max:1024M dom0_max_vcpus=2 conswitch=x dtuart=/soc/uart@f7113000 altp2m=1
 kernel=Image console=hvc0 root=/dev/mmcblk1p2 rootwait rw
 dtb=hi6220-hikey.dtb
 ```

 * Config File: `startup.nsh`

 ```bash
 vim $EFI/xen.cfg

  FS0:
  Image console=ttyAMA3,115200 dtb=hi6220-hikey.dtb root=/dev/mmcblk1p2 rootwait rw
 ```

Finally unmount the `EFI partition` and exit the root shell.


```bash
umount $EFI
sync

exit
```


## **Cross-Compiling Xen-tools**

While the the user space components of Xen (including libraries and management tools) can be compiled natively on the ARM development board, we can make use of the system's cross-compiling capabilities to ease the entire process. Similar to the development of the Xen hypervisor itself, this process becomes especially relevant if you develop not directly on the board. More information regarding cross-compilatoin can be found on the [Xen's cross-compiling page](https://wiki.xenproject.org/wiki/Xen_ARM_with_Virtualization_Extensions/CrossCompiling#Build_arm64_tools "Cross-compiling Xen")

To cross-compile Xen-tools on your host, first, we have to make sure that we have all needed packages:

```bash
sudo apt install sbuild
```

Then, we have to prepare a **chroot** environment:

```bash
sudo sbuild-adduser <your_username>
sudo sbuild-createchroot <debian_release> /srv/chroots/<debian_release>-arm64-cross <debian_http_mirror>
mv /etc/schroot/chroot.d/<debian_release>-amd64-sbuild-* /etc/schroot/chroot.d/<debian_release>-arm64-cross
```

Make sure that the file `/srv/chroots/<debian_release>-arm64-cross` contains at least the following entires:

```bash
vim /srv/chroots/<debian_release>-arm64-cross

[<debian_release>-arm64-cross]  
description=Debian <debian_release>/arm64 crossbuilder
groups=root,sbuild
root-groups=root,sbuild
profile=default
type=directory
directory=/srv/chroots/<debian_release>-arm64-cross
```

To switch into the **schroot** environment, we use the following command:

```bash
sudo schroot -c <debian_release>-arm64-cross
```

Once inside the **schroot** environment, we have to meet all dependencies:

```bash
apt-get update
apt install vim-tiny wget sudo less
su <username>
sudo dpkg --add-architecture arm64
sudo apt-get update
sudo apt install crossbuild-essential-arm64 libc6-dev:arm64 libncurses-dev:arm64 uuid-dev:arm64 libglib2.0-dev:arm64 libssl-dev:arm64 libssl-dev:arm64 libaio-dev:arm64 libyajl-dev:arm64 python gettext gcc git libpython2.7-dev:arm64 libfdt-dev:arm64 iasl:arm64 libpixman-1-dev:arm64 autotools-dev
cp /usr/share/misc/config.{sub,guess} .
```

Then, we can configure and cross-compile Xen-tools:

```bash
CONFIG_SITE=/etc/dpkg-cross/cross-config.arm64 ./configure --build=x86_64-unknown-linux-gnu --host=aarch64-linux-gnu
make dist-tools CROSS_COMPILE=aarch64-linux-gnu- XEN_TARGET_ARCH=arm64 -j7
exit        # exit form your user chrooted-space
exit        # exit form the chroot environment
```

Finally, we can install Xen-tools on the Dom0 partition:

```bash
sudo su

export DOM0=/mnt/dom0

mount /dev/sdx2 $DOM0

rsync -avpI ./dist/install/ $DOM0

exit
```

  ---
  ### **NOTE:** Remote Intallation of Xen-Tools

  Xen-tools can be installed from remote via `SSH`:

  ```bash
    rsync -avpI -e ssh <destination_host> "./dist/install/*"
  ```
  ---

If the Xen-tools are installed for the first time, you should configure services required by Xen to be enabled on system startup in Dom0:

```bash
chroot $DOM0
update-rc.d xencommons defaults 19 18
update-rc.d xendomains defaults 21 20
update-rc.d xen-watchdog defaults 23 22
ldconfig

exit

umount $DOM0
sync
```


## **Booting the Lemaker Hikey Development Board**

The Lemaker Hikey development is an UEFI-based system. It must be equipped with a serial module that we can use for establishing a communication through the serial port. For this, we leverage the serial console provided by `minicom` on our host system. To startup Xen on the board (with the previously defined configuration for its domains) we have to follow the following steps:

```bash
sudo minicom -b 115200 -D /dev/ttyUSB0      
```

After having established the communication, start up the board:

1. Press `<enter>` - to suppress the message `The default boot selection will start in  10 seconds`
1. Jump into the EFI-Shell. In our case, type `4` and press `enter`
1. Press `esc` repeatedly until getting access to `Shell` - this is only needed if we don't want to boot according to `startup.nsh` (in our case `startup.nsh` boots up the Linux kernel instead of Xen by default).
1. type `xen` and press enter - this will look for a `xen.cfg` and will load `xen.efi`


## **Configuring DomU**

The unprivileged domain DomU can be set up similarly to Dom0. The following configuration script can be used to startup the domain:

```bash
kernel = "<path_to_kernel_image>/Image"
acpi = 1
memory = 1024
name = "domu1"
altp2m = "external"
on_poweroff = "destroy"
on_reboot = "destroy"
on_crash = "destroy"
vcpus = 2
disk = [ 'phy:/dev/<path-to-domu-dev>,xvda,w' ]
extra = "earlyprintk=xenboot console=hvc0 root=/dev/xvda debug rw init=/sbin/init"
```

## **DRAKVUF and LibVMI**

As the [LibVMI](http://LibVMI.com/ "LibVMI") library by now implements sufficient support that is required for stealthy monitoring on ARM, the code can be cloned directly from the associated [Github repository](https://github.com/LibVMI/LibVMI "LibVMI Github").

To get the most recent sources for [DRAKVUF](https://drakvuf.com/ "DRAKVUF Homepage"), we would like to refer to the pending [pull request](https://github.com/tklengyel/drakvuf/pull/445 "ARM support for DRAKVUF").

Both required configuration steps for LibVMI and DRAKVUF and a brief howto describing the usage of basic DRAKVUF capabilities on ARM can be found on the following [blog post](http://arm-drakvuf.blogspot.com/ "Support for DRAKVUF on ARM").

## **Generating the Linux Kernel DWARF Object**

  When starting DRAKVUF to monitor a guest, the framework needs a profile of the guest's underlying kernel in order to extract information such as addresses of various objects. The profile can be created using Google's [rekall](https://github.com/google/rekall) tool in conjunction with a tool built by the authors to complement the lacking features of rekall for Linux kernel for ARM's Aarch64. In order to build the DWARF object, first clone the rekall tool from the official repository and modify the content of `<path_to_rekall>/tools/linux/Makefile` to the following content:

    ```
    obj-m += module.o                                                                   
    obj-m += pmem.o                                                                     

    KVER ?= <linux_version>-<your_username>-v<current_version>
    KHEADER ?= <path_to_linux_kernel>                           
    KSYSTEMMAP ?= <path_to_linux_kernel>/System.map              
    KCONFIG ?=  <path_to_linux_kernel>/.config

    # Fedora Core                                                                       
    # KHEADER ?= /lib/modules/$(KVER)/source

    # This needs to be specified because when running make under sudo we have no        
    # access to the PWD environment var.                                                
    PWD = `pwd`

    -include version.mk

    #CFLAGS_jc.o := -g0

    all: dwarf pmem profile

    pmem: pmem.c                                                                        
          $(MAKE) -C $(KHEADER) M=$(PWD) modules                                      
          cp pmem.ko "pmem-$(KVER).ko"                                                

    dwarf: module.c                                                                     
           $(MAKE) -C $(KHEADER) CONFIG_DEBUG_INFO=y M=$(PWD) modules                  
           cp module.ko module_dwarf.ko                                             

    profile: dwarf pmem                                                                 
             zip "$(KVER).zip" module_dwarf.ko $(KSYSTEMMAP) $(KCONFIG) "pmem-$(KVER).ko"

    clean:                                                                              
            $(MAKE) -C $(KHEADER) M=$(PWD) clean                                        
            rm -f module_dwarf.ko
    ```

    Execute `make dwarf` to compile the DWARF object.

## **Generating the Linux Kernel Profile**

  After the DomU object is generated, the repository containing the scripts to the [rekall-profile-generator](https://github.com/drakvuf-on-arm/rekall-profile-generator) must be cloned. In the repository's local directory execute the following commands to create the kernel's profile:

  `./rekall-profile-generator.py -s <path_to_linux_kernel>/System.map -c <path_to_linux_kernel>/.config -d <path_to_rekall>/tools/linux/module_DomU.ko`

  You need to copy the generated `profile` in the SD-Card that will be used by the ARM board.

## **Configuring LibVMI**
  This section describes the steps required to compile and install LibVMI on the ARM board. First, one needs to install the library dependencies according to `README.rst` from the LibVMI repository.

  ```bash
  ./autogen.sh
  ./configure
  make -j
  make -j install
  ldconfig
  ```

  After successfully compiling and installing LibVMI, one needs to create a LibVMI profile for the kernel that is analyzed. Therefore, a couple of kernel parameters are needed by LibVMI which can be extracted by inserting in the guest kernel a driver provided by the LibVMI developers. In the the LibVMI local repository follow the steps in the `tools/linux-offset-finder/README` file to learn how to compile and insert the module.

## **Configuring DRAKVUF and Attaching to a DomU Guest**
  This section presents the compilation steps and the execution arguments that the user can choose to run [DRAKVUF](https://github.com/drakvuf-on-arm/drakvuf) and attach it to an already executing DomU guest. Also, one needs to install the dependencies that DRAKVUF requires which are listed in the `README` of the repository.
  ```bash
  cd <path_to_drakvuf>
  ./autogen.sh
  ./configure --disable-plugin-poolmon --disable-plugin-filetracer --disable-plugin-filedelete --disable-plugin-objmon --disable-plugin -exmon --disable-plugin-ssdtmon --disable-plugin-debugmon --disable-plugin-cpuidmon --disable-plugin-socketmon --disable-plugin-regmon --disable-plugin-procmon --enable-debug
  make -j
  src/drakvuf -r <path_to_rekall_profile>/profile -d <name_of_DomU> -a <option> # this command start drakvuf; OPTIONAL: add `-v` to enable debugging output;
  ```

## **About the Authors**

  [Sergej Proskurin](mailto:proskurin@sec.in.tum.de) and [Marius Momeu](mailto:momeu@sec.in.tum.de) are researchers at the [Chair of IT Security](https://www.sec.in.tum.de/i20/) at the [Technical University of Munich](https://www.tum.de/). Their work covers many low-level security aspects with a focus on Malware Analysis using Virtual Machine Introspection (VMI). More information about previous publications and projects as well as current undergoing projects can be found on the authors' academic [web page](https://www.sec.in.tum.de/i20/people/sergej-proskurin).

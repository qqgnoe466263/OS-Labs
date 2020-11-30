# Embeded system development with QEMU
###### tags: `ARM` `Uboot` `QEMU` 

## Development environment

* Host
    * Ubuntu 18.04.1 x86_64
    * Linux ubuntu 5.3.0-46-generic
    * Intel(R) Core(TM) i7-3770 CPU @ 3.40GHz
    * DRAM DDR3 16GB
* Guest
    * Linux-4.14.13
    * busybox-1.31.0
    * QEMU emulator version 2.11.1
    * u-boot-2019.10
    * gdb-8.1 (重編譯成ARM架構的)
* Directory Hierarchy (~/embs/)
    * busybox-1.31.0
    * linux-4.14.13
    * sd_card
    * u-boot-2019.10 
    * gdb-8.1

## Prerequisites

```bash=
$ sudo apt-get install qemu qemu-system -y
$ sudo apt-get install libncurses5-dev build-essential -y
$ sudo apt-get install gcc-arm-linux-gnueabihf -y
$ sudo apt-get install flex -y
```
* `qemu` : 是一個免費的虛擬機軟體，QEMU的架構由純軟體實現，並在Guest與Host中間，來處理Guest的硬體請求，並由其轉譯給真正的硬體。
* `libncurses5-dev` : 是一個`ncurses`開發函式庫，可以允許程式設計師編寫獨立於終端的基於文字的使用者介面。
* `build-essential` : 提供必要的編譯環境。
* `gcc-arm-linux-gnueabihf` : 提供GCC ARM Cross Compiler Toolchain。
* `flex ` : 提供語意分析(編譯會用到)。

## Linux kernel (ARM)

### Linux kernel source code (v4.14.13)

```bash=
$ wget https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/linux-4.14.13.tar.gz 
$ tar -xvf linux-4.14.13.tar.gz 
```

### Compiler kernel (ARM)

```bash=
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- vexpress_defconfig # create a vexpress config file
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig # GUI setting config
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage -j8 
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- dtbs
```

* `ARCH` : 硬體架構選擇**arm**
* `CROSS_COMPILE` : 交叉編譯工具選擇**arm-linux-gnueabihf-**
* `zImage` : ***ARM Linux***常用的壓縮映像檔。(gzip壓縮)
* `dtbs` : `Device Tree`格式為`dts`,但`uboot`和`linux`不能讀,所以把`dts`編譯成`dtb`。

### Launch up

```bash=
$ cd linux-4.14.13/
$ qemu-system-arm -M vexpress-a9 -m 512M -kernel arch/arm/boot/zImage -dtb arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic -append "console=ttyAMA0"
```
> But you will see `kernel panic`,because kernel can't find `root file system`

* `-M vexpress-a9` : 模擬`vexpress-a9`開發板
* `-m 512M` : 記憶體設定成`512MB`
* `-kernel <path>` : 給qemu啟動kernel的路徑
* `-nographic` : 不使用GUI，使用`tty`
* `-append` : 給`kernel`啟動參數
    * `console=ttyAMA0` : 告訴kernel要用哪個terminal。(.config中的CONFIG_CONSOLE設定)


> `tty` : 在linux中是terminal的統稱，`tty1~6`是`cmd line`介面，`tty7`是`GUI`介面


## Root filesystem

* Root filesystem = busybox(basic linux cmd) + libary + tty(terminal)

### Busybox source code

```bash=
$ wget https://busybox.net/downloads/busybox-1.31.0.tar.bz2
$ tar -xvf busybox-1.31.0.tar.bz2
```

### Compile Busybox

```bash=
$ cd busybox-1.31.0/
$ make menuconfig
    Settings -> Build Options -->
        [*] Build Busybox as a static binary
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j8
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j8 install
```

### Create Root file system

```bash=
$ mkdir -p rootfs
$ mkdir -p rootfs/dev
$ mkdir -p rootfs/etc/init.d
$ mkdir -p rootfs/lib
$ cp -r ./_install/* rootfs/

# copy library to lib/ from cross-tool-chain
$ sudo cp -p /usr/arm-linux-gnueabihf/lib/* rootfs/lib/

# create 4 tty 
$ sudo mknod rootfs/dev/tty1 c 4 1
$ sudo mknod rootfs/dev/tty2 c 4 2
$ sudo mknod rootfs/dev/tty3 c 4 3
$ sudo mknod rootfs/dev/tty4 c 4 4

# create rootfs image
$ dd if=/dev/zero of=a9rootfs.ext4 bs=1M count=32
# format -> ext3 file system
$ mkfs.ext4 a9rootfs.ext4

# copy files to image
$ sudo mkdir tmpfs
$ sudo mount -t ext4 a9rootfs.ext4 tmpfs/ -o loop
$ sudo cp -r rootfs/*  tmpfs/
$ sudo umount tmpfs

# create file system done !
```

### Launch up

```bash=
$ qemu-system-arm -M vexpress-a9 -m 512M -kernel arch/arm/boot/zImage -dtb arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic -append "root=/dev/mmcblk0 console=ttyAMA0" -sd ../busybox-1.31.0/a9rootfs.ext4
```

* `root=/dev/mmcblk0` : `root=`是指說指定`root file system`的開機位置。(mmc是SD卡的前身，SD是根據MMC的產品規格及特性所研發出來的，因此適用SD的記憶卡插槽MMC也適用)

## U-boot

### U-boot source code

```bash=
$ wget ftp://ftp.denx.de/pub/u-boot/u-boot-2019.10.tar.bz2
$ tar -xvf u-boot-2019.10.tar.bz2
```

### Compile U-Boot

```bash=
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- vexpress_ca9x4_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j8
```

## Make a SD card

```bash=
$ mkdir sd_card
$ cd sd_card

# create an empty image
$ dd if=/dev/zero of=uboot.disk bs=1M count=1024

# sgdisk - Command-line GUID partition table (GPT) manipulator for Linux 
# one for kernel and device tree, one for root file system
$ sgdisk -n 0:0:+10M -c 0:kernel uboot.disk
$ sgdisk -n 0:0:0 -c 0:rootfs uboot.disk

# find the first unused loop device
$ losetup -f 
/dev/loop3 # In my system,I got this.

# mapping my SD card to loop device
$ sudo losetup /dev/loop3 uboot.disk
$ sudo partprobe /dev/loop3

# low level formatting
$ sudo mkfs.ext4 /dev/loop3p1
$ sudo mkfs.ext4 /dev/loop3p2

$ mkdir p1 p2

# Mount
$ sudo mount -t ext4 /dev/loop16p1 p1/
$ sudo mount -t ext4 /dev/loop16p2 p2/

# Copy files to SD card 
$ sudo cp ../linux-4.14.13/arch/arm/boot/zImage p1/
$ sudo cp ../linux-4.14.13/arch/arm/boot/dts/vexpress-v2*.dtb p1/
$ sudo cp -raf ../busybox-1.31.0/rootfs/* ./p2

# umount
$ sudo umount p1 p2
$ sudo losetup -d /dev/loop3
```

### Start up (u-boot)

```bash=
$ qemu-system-arm -M vexpress-a9 -m 512M -nographic -kernel ./u-boot -sd ./uboot.disk

=> load mmc 0:1 0x60008000 zImage
=> load mmc 0:1 0x61000000 vexpress-v2p-ca9.dtb
=> setenv bootargs 'root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait earlycon console=tty0 console=ttyAMA0 init=/linuxrc printk.time=y initcall_debug'
=> bootz 0x60008000 - 0x61000000
```

## ARM gdb

* if you use standard gdb,you will get this bug `Truncated register 16 in remote 'g' packet`.

* So we need `ARM gdb`.

```bash=
$ wget http://ftp.gnu.org/gnu/gdb/gdb-8.1.tar.gz
$ tar -xvf gdb-8.1.tar.gz
$ cd gdb-8.1
$ ./configure --target=arm-linux
$ make -j8
```
* Debug script

```bash=
$ vim debug.sh

#!/bin/sh
../gdb-8.1/gdb/gdb --data-directory ../gdb-8.1/gdb/data-directory ./u-boot \
    -ex "target remote : 1234" \
    -ex "b board_init_f"
```

## U-boot booting ~

* 這邊列出一些重要的階段

    _start -> board_init_f -> relocate_code -> board_init_r -> main_loop

## U-boot debug Note (symbol missing)

* 跟到這一步的時候下一步會跳到`relocaddr`，因為uboot會跳進RAM中所以會relocaddr，總之跳進去後`symbol`會跑掉，所以要給gdb新的`symbol`。

![](https://i.imgur.com/84AmLKS.jpg)

```script=
(gdb) symbol-file
Discard symbol table from `/home/pwn/embs/u-boot-2019.10/u-boot'? (y or n) y
Error in re-setting breakpoint 1: No symbol table is loaded.  Use the "file" command.
Error in re-setting breakpoint 2: No source file named /home/pwn/embs/u-boot-2019.10/arch/arm/lib/relocate.S.
No symbol file now.
(gdb) add-symbol-file u-boot 0x60f8a000
add symbol table from file "u-boot" at
        .text_addr = 0x60f8a000
(y or n) y
Reading symbols from u-boot...done.
(gdb)
```

* 重設後`symbol`就回來了。

![](https://i.imgur.com/ZjRF2aa.jpg)


## Measure Boot Time Note

> 由這個去追，可以知道他time怎麼算的 (一個tick 0.001秒?)
* https://stackoverflow.com/questions/20337221/how-can-i-access-the-hardware-time-from-u-boot
* https://blog.csdn.net/ooonebook/article/details/53164198

> 一些感覺可以加入偵測的地方(?)
```bash=
// relocaddr
0x60f8a000

include/initcall.h
	initcall_run_list

// 第一階段 初始化
common/board_f.c 

// 第二階段 初始化
common/board_r.c
    board_init_r  <- 進入初始化
    run_main_loop <- 最後跑進這裡 

common/command.c

cmd/time.c
```
## Measure Boot Time Note2

* Method1 : 將輸出導向到某個`serial port`然後算一下開始與結束，可以用`grabserial`工具，但都導向不成功orz。
* Method2 : `QEMU`可以導到TCP上面，那就寫個`py script`抓個字串測量時間就好了，但可能不太準要考慮到網路blabla的效率?

```bash=
$ qemu-system-arm -M vexpress-a9 -m 16M -nographic -kernel ./u-boot -append "console=ttyAMA0" -serial tcp:127.0.0.1:4444
```
```python=
#!/usr/bin/env python                                                     
from pwn import *
import time
l = listen(4444)

s = time.time()

while True:
    r = l.recvline()
    if "ERROR: can't get kernel image!" in r : 
        break

e = time.time()
print "[{}]".format(e - s)

l.close()
```

## Decrease boot time

* common/main.c

```c=
void main_loop()
{
    ...
    // autoboot_command(s); 
    ...
}
```
* common/autoboot.c

```c=
void autoboot_command(const char *s) 
{
    debug("### main_loop: bootcmd=\"%s\"\n", s ? s : "<UNDEFINED>");

    if (stored_bootdelay != -1 && s && !abortboot(stored_bootdelay)) {
        bool lock;
        int prev;

        lock = IS_ENABLED(CONFIG_AUTOBOOT_KEYED) &&
            !IS_ENABLED(CONFIG_AUTOBOOT_KEYED_CTRLC);
        if (lock)
            prev = disable_ctrlc(1); /* disable Ctrl-C checking */

        run_command_list(s, -1, 0); /* 做一大堆的初始化的什麼東東，找mmc device找kernel image...等等 */

        if (lock)
            disable_ctrlc(prev);    /* restore Ctrl-C checking */
    }

    if (IS_ENABLED(CONFIG_USE_AUTOBOOT_MENUKEY) &&
        menukey == AUTOBOOT_MENUKEY) {
        s = env_get("menucmd");
        if (s)
            run_command_list(s, -1, 0);
    }
}        
```

* common/cli_hush.c


## Add a new cmd

* cmd/hello.c

```c=
#include <common.h>
#include <command.h>

static int do_hello(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
    printf("hello world\n");
    return 0;
}

U_BOOT_CMD(hello, CONFIG_SYS_MAXARGS, 0, do_hello,
        "hello : print hello world\n",
        "No arg ... \n");
```

* cmd/Makefile

```bash=
...
# command
obj-y += hello.o
...
```




## Ref

* https://blog.csdn.net/linyt/article/details/42504975
* https://www.cnblogs.com/pengdonglin137/p/12194548.html
* https://blog.csdn.net/qq_29729577/article/details/52062722
* https://blog.csdn.net/u012489236/article/details/97137007?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase
* https://b8807053.pixnet.net/blog/post/340302750-%5B%E8%BD%89%5Dlinux-device-tree--1-%E8%B5%B7%E6%BA%90
* https://stackoverflow.com/questions/25597445/python-exception-type-exceptions-importerror-no-module-named-gdb
* http://www.denx.de/wiki/view/DULG/DebuggingUBoot#Section_10.1.2
* http://albert-oma.blogspot.com/2016/07/embedded-u-boot.html
* https://blog.csdn.net/ooonebook/article/details/53164198
* https://www.cnblogs.com/pengdonglin137/p/3328384.html
* https://draapho.github.io/2017/08/30/1721-uboot-modify/

# Compile Linux Kernel && Explain : how to Tranlate virtual address to physical address
###### tags: `Linux Kernel` `virtual address` `physical address`

## Development environment

* Host
    * Ubuntu 20.04 x86_64
    * Linux ubuntu 5.8.0-63-generic
    * Intel(R) Core(TM) i5-8500 CPU @ 3.00GHz
    * DRAM DDR4 8GB
* Guest
    * Linux-5.5.1 x86
    * minimal Debian Linux image (file system) (from google)
    * QEMU emulator version 2.11.1
* Directory Hierarchy (~/NCU_LINUX/):

```bash
# Initial directory hierarchy
~/NCU_LINUX/
    - linux-5.5.1
    - create-image.sh # creates a minimal Debian Linux image suitable for syzkaller from google
```
```bash
# Full directory hierarchy
~/NCU_LINUX/
    - linux-5.5.1
    - chroot
        - <testing program>
    - stretch.img # file system image
    - launch.sh # launch virual machine script
    - mkimg.sh  # create fs img script
    - create-image.sh 
```

---

## Prerequisites

```bash
$ sudo apt-get install qemu qemu-system -y
$ sudo apt-get install libncurses5-dev build-essential -y
$ sudo apt-get install flex bison -y
$ sudo apt-get install bridge-utils -y
$ sudo apt-get install gcc-multilib -y # for gcc -m32
$ sudo apt-get install debootstrap -y
$ sudo apt-get install libssl-dev -y
```

## Linux kernel (x86_64)

### Linux kernel source code (v5.5.1)

```bash
$ wget https://www.kernel.org/pub/linux/kernel/v5.x/linux-5.5.1.tar.gz
$ tar -xvf linux-5.5.1.tar.gz
```

### Configuration 
> Before compiling kernel, can choose some configs need or not
```bash
$ cd linux-5.5.1/
# method 1
$ make menuconfig # GUI 
# method 2
$ vim .config 
```

* Method1

```bash
[ ] 64-bit kernel # choose x86 or x64 system (x64 by default)

KernelHacking -->
    [*]Compile the kernel with debug info
    [ ]Tracers
    x86 Debugging --->
        [*] Debug low-level entry code

# If you want to compile faster, can disable some drivers
# 測試中
Device Drivers -->
    [ ] Multiple devices driver support (RAID and LVM) --->
    Network device support ---> # Actaully we can disable all configs if we wantn't to use network
        [ ] Network core driver support
        [ ] Wireless LAN --->
    [ ] Sound card support
    [ ] USB support
    Android -->
        [ ] Android Drivers

# This make debugging easier
Processor type and features -->
	Build a relocatable kernel
		[ ] Randomize the address of the kernel image (KASLR)
```

* Method2

```bash
# deny these features if ur host is ubuntu20.04
# CONFIG_MODULE_SIG_ALL
# CONFIG_MODULE_SIG_KEY
# CONFIG_SYSTEM_TRUSTED_KEYS
```


### Compile kernel 
```bash
$ cd linux-5.5.1
$ make -j8
$ make -j8 bzImage
```
> After compiling kernel, we will get this `linux-5.5.1/arch/x86/boot/bzImage`

## Root File system

* Create root file system.
```bash
$ wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh -O create-image.sh
$ chmod +x create-image.sh
$ ./create-image.sh -a i386
# After this, we will get 2 important files chroot stretch.img
$ ls 
chroot stretch.img linux-5.5.1...
```

## Launch up 

```bash
$ vim launch.sh

#!/bin/bash
sudo qemu-system-x86_64 \
  -kernel ./linux-5.5.1/arch/x86/boot/bzImage \
  -append "console=ttyS0 root=/dev/sda earlyprintk=serial nokaslr"\
  -hda ./stretch.img \
  -nographic \
  -m 4G \
#  -s -S # This is for gdb debug
# -monitor tcp:127.0.0.1:4444,server,nowait # This is for qemu-monitor
```
```bash
# Now we check our work dir, should have these files.
$ ls
chroot linux-5.5.1 stretch.img launch.sh ...
$ chmod +x launch.sh
# And we can launch our kernel now ~
$ ./launch.sh
```

## Proof of Work

* 在QEMU中可用

```bash
(gdb) lx-iomem
...
01000000-01a589c3 : Kernel code
01a59000-01e58fff : Kernel rodata
01e59000-01f3a3ff : Kernel data
02065000-0214efff : Kernel bss
...
```

* 實際上要加上 0xc0000000 才是能看到東西

```bash
# 例如
(gdb) x/10gx 0xc1000000
0xc1000000 <startup_32>:        0x86f601e5ed600d8b      0x0f16754000000211
0xc1000010 <startup_32+16>:     0x18b801e5ed661501      0x8ec08ed88e000000
0xc1000020 <startup_32+32>:     0x00a18dd08ee88ee0      0x00bfc031fc400000
```

* virtual 2 physical
	* Demo case : kernel virtual address

```bash
(gdb) p $init_mm.pgd
	pgd : 0xc2067000

0xc1000000 -> 轉成bits來看
   |  10bits  |  10bits  |  12bits    |
    1100000100 0000000000 000000000000
	
level-1 entry = 0xc2067000 + (1100000100 << 2) (前10bit左移2)
              = 0xc2067000 + 772 * 4
            
# 用gdb看一下裡面的值 (0x1e1這個通常是一些 write read 的 flag 我們不看直接可以省略)
(gdb) x/wx 0xc2067000 + 772 * 4
0xc2067c10:     0x010001e1

level-2 entry = 0x01000000 + (0000000000 << 2) (中間10bit左移2)
              = 0x01000000
               
# 可是在 gdb 不能存取 0x10000000 (因為這是實體記憶體位置) (可用qemu-monitor驗證)
(gdb) x/10gx 0x1000000
0x1000000:      Cannot access memory at address 0x1000000

# *0x1000000 + 最後12bits = 實體記憶體資料的地方
```

* 由上我們可以得知，v2p是有成功的，因為`0xc1000000`->`0x01000000`是正確的。

## Appendix : How to use gdb to debug kernel

* In your work dir

```bash
$ ls # 你的工作目錄下應該要有這些
chroot linux-5.5.1 ....
# 先設定一下 .gdbinit
$ vim ~/.gdbinit
# 加入下面四條
# 請更換 <your work_dir> 成你的路徑
add-auto-load-safe-path <your work_dir>/linux-5.5.1/scripts/gdb/vmlinux-gdb.py
set auto-load safe-path /
set architecture i386:x86-64
target remote:1234

# 把你 launch.sh 的 -s -S 的 註解 拿掉
# 然後
$ ./launsh.sh
# 開另外一個 terminal
$ gdb linux-5.5.1/vmlinux
....
(gdb) c # continue
```

### gdb script bug

* 此類`task`功能(e.g. `lx_task_by_pid`)，有bug，讓功能正常的方式為:
    * 修一下這個`script`

```bash
$ vim linux-5.5.1/scripts/gdb/linux/task.py
```
* 隨便的修法

```python

# 隨便寫一個爬task struct的function
def my_task_lists(pid):
    task_ptr_type = task_type.get_type().pointer()
    init_task = gdb.parse_and_eval("init_task").address
    g = t = init_task
    t = utils.container_of(t['thread_group']['next'], task_ptr_type, "thread_group")
    t = g = utils.container_of(g['tasks']['next'], task_ptr_type, "tasks")
    while g['pid'] != pid:
        t = g = utils.container_of(g['tasks']['next'], task_ptr_type, "tasks")
    return t

# 改一下這個function
def get_task_by_pid(pid):
    return my_task_lists(pid)
```

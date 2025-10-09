Guides used:
https://vatlidak-org.github.io/web/assets/md/setup_instructions/ <br>
https://github.com/heechul/palloc

<br>

---

<h1 style="font-size: 2.5em;">CONFIG / BUILD / INSTALL / BOOT</h1>

# Setting up a QEMU+KVM VM
Already done: <br>
→ given 2 Cores, 8GB of RAM​, 120GB of SSD (60GB available for PALLOC, rest used by old EXTMEM). <br>
→ Ubuntu 22.04 minimal install​. <br>

My Ubuntu VM has already these kernel available for boot:
```
/boot/vmlinuz-6.8.0-59-generic   # original default
/boot/vmlinuz-6.8.0-60-generic   # original default
/boot/vmlinuz-5.15.0+           	# patched kernel 5.15 with EXTMEM (old)
```

# Kernel Compilation & Installation:
For PALLOC I will use the palloc-6.6.patch. So, firstly a kernel version 6.6 must be downloaded, compiled, installed, booted. After that, it will be patched with palloc.

## Preparation
Boot up the VM and from a shell inside the VM execute the following commands to install the required dependencies for building the kernel:
```
$ sudo apt update
$ sudo apt install build-essential git bc python3 bison flex rsync libelf-dev libncurses-dev dwarves libdw-dev gawk libssl-dev
```

Next, clone the source code for Linux kernel v6.6:
```
$ git clone --depth=1 --branch v6.6 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git linux-6.6
$ cd linux-6.6
```

Warning: all subsequent commands from this guide should be run from the root directory of the kernel source!   <br>
Read the first lines of the Makefile to verify you downloaded the correct kernel version:
```
$ head -n5 Makefile
```
```
# SPDX-License-Identifier: GPL-2.0
VERSION = 6
PATCHLEVEL = 6
SUBLEVEL = 0
EXTRAVERSION =
```

## Kernel Configuration
Before building the kernel, you must configure it by selecting the features, drivers and other options that will be included in the compiled kernel. This configuration is stored in the .config file in the root directory of the kernel source.

We will remove any existing configuration files and create our own based on the configuration file of the kernel currently in use.

Warning: it would be a good idea to run this step using a stable stock kernel, such as the one provided by default by your distribution, to ensure you proceed with a proper configuration.
```
$ make mrproper
$ make olddefconfig
```

A .config file should now be visible in the current directory. This file holds kernel build options, each set to a specific value. You can view its contents with cat .config. There are two main approaches to edit the configuration file:
   - Run make menuconfig to use an interactive menu, or <br>
   - Use the scripts/config tool: <br>
```
scripts/config --set-str <option> <value>       # set an option
scripts/config --state <option>                 # read an option
```

Make the following changes to your configuration file: 
```
$ scripts/config --set-str CONFIG_LOCALVERSION "-palloc"
$ scripts/config --set-str SYSTEM_TRUSTED_KEYS ""
$ scripts/config --set-str SYSTEM_REVOCATION_KEYS ""
```
The first command gives your kernel a custom name to distinguish it from other installed kernels. You should expect your custom kernel to have a name similar to 6.6.0-palloc. The latter commands instruct the kernel not to enforce built-in signature verification of modules/firmware against specific trusted keys.

## Kernel Build
Build the kernel using all cores allocated to the VM with:
```
$ make -j`nproc`
```
Depending on the amount of available cores and the capabilities of your machine, this step may take a while.

## Kernel Installation
Install your custom kernel and kernel modules with:
```
$ sudo make modules_install && sudo make install
```

If all goes well, you must be able to verify that the following 4 files are in /boot:
```
$ ls /boot | grep "palloc"
config-6.6.0-palloc
initrd.img-6.6.0-palloc
System.map-6.6.0-palloc
vmlinuz-6.6.0-palloc
```

## Booting the Custom Kernel
Before rebooting the machine, we need to modify the bootloader’s (GRUB) configuration file to allow us to select the kernel we want to boot with.

Open /etc/default/grub as root with a text editor and: <br>
   - Replace GRUB_TIMEOUT_STYLE=hidden with GRUB_TIMEOUT_STYLE=menu <br>
   - Replace GRUB_TIMEOUT=0 with GRUB_TIMEOUT=15 <br>

Save and close the file and then run:
```
$ sudo update-grub
```

Reboot the machine and from the GRUB menu select Advanced Options for Ubuntu . Choose the kernel post-fixed with -palloc. You can always select the Ubuntu option from the GRUB menu to boot with the stock kernel, which is recommended for development.

Once the VM boots up, verify you are running your custom kernel with:
```
$ uname -r
6.6.0-palloc
```

# Kernel Patch with PALLOC

## Get the PALLOC patch
Go to https://github.com/heechul/palloc, where the palloc-6.6.patch file is what we need.
To get this, use wget inside the linux-6.6 (kernel) directory: 
```
$ cd /home/teomorvm/Desktop/PALLOC-kernel-6.6/linux-6.6
$ wget https://raw.githubusercontent.com/heechul/palloc/master/palloc-6.6.patch
```

## Apply the PALLOC patch
```
$ patch -p1 < palloc-6.6.patch
```
```
patching file include/linux/cgroup_subsys.h
patching file include/linux/mmzone.h
Hunk #2 succeeded at 940 (offset -4 lines).
patching file include/linux/palloc.h
patching file init/Kconfig
Hunk #1 succeeded at 1175 (offset -9 lines).
patching file kernel/cgroup/cgroup.c
Hunk #1 succeeded at 6045 (offset 12 lines).
patching file mm/Makefile
patching file mm/mm_init.c
Hunk #1 succeeded at 1378 (offset -6 lines).
patching file mm/page_alloc.c
Hunk #3 succeeded at 1743 (offset -21 lines).
Hunk #4 succeeded at 2107 (offset -21 lines).
Hunk #5 succeeded at 3287 (offset -22 lines).
Hunk #6 succeeded at 3295 (offset -22 lines).
Hunk #7 succeeded at 6929 (offset -24 lines).
patching file mm/palloc.c
patching file mm/vmstat.c
```

Enable and verify this:
```
$ scripts/config --enable CONFIG_CGROUP_PALLOC
$ CONFIG_CGROUP_PALLOC=y
$ grep PALLOC .config
```

## Re-build and Re-install the patched kernel
```
$ make -j`nproc`
$ sudo make modules_install && sudo make install
```

Note: You do not need to re-configure the kernel every time you make a change. However you should re-compile any changed files and re-install the modified kernel using the commands above whenever you are ready to test your changes.

Reboot the machine and once again select your custom -palloc kernel from the GRUB menu. The new kernel may show a -dirty suffix, which is normal:
```
$ uname -r
6.6.0-palloc-dirty
```

So, my Ubuntu VM now has these kernels available for boot:
```
/boot/vmlinuz-6.8.0-59-generic   # original default
/boot/vmlinuz-6.8.0-60-generic   # original default
/boot/vmlinuz-5.15.0+       	   # patched kernel 5.15 with EXTMEM (old)
/boot/vmlinuz-6.6.0-palloc		   # original kernel 6.6
/boot/vmlinuz-6.6.0-palloc-dirty	# patched kernel 6.6 with PALLOC
```

<br>

---

<h1 style="font-size: 2.5em;">FUNCTION / EXPERIMENT / REPRODUCE / RESULTS</h1>


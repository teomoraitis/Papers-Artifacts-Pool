# Notes-Progress on Coloring support via PALLOC+patch usage
Guides used: <br>
https://github.com/heechul/palloc <br>
https://vatlidak-org.github.io/web/assets/md/setup_instructions/ <br>
https://docs.kernel.org/process/applying-patches.html <br>

PALLOC is a kernel-level memory allocator that exploits page-based virtual-to-physical memory translation to selectively allocate memory pages of each application to the desired DRAM banks. The goal of PALLOC is to control applications' memory locations in a way to minimize memory performance unpredictability in multicore systems by eliminating bank sharing among applications executing in parallel. PALLOC is a software based solution, which is fully compatible with existing COTS hardware platforms and transparent to applications (i.e., no need to modify application code). 
PALLOC is a color-aware page-based physical memory allocator which replaces the buddy allocator for order-zero page allocations.

<br>

---

# Table of Contents

1. **PALLOC Config / Build / Install / Boot**
   - [Setting up a QEMU+KVM VM](#setting-up-a-qemukvm-vm)
   - [Kernel Compilation & Installation](#kernel-compilation--installation)
   - [Kernel Patch with PALLOC](#kernel-patch-with-palloc)

2. **PALLOC Functionality / Experiment / Reproduce / Results**
   - [Functionality](#functionality)
   - [Experiment](#experiment)
   - [Reproduce-Results](#reproduce-results)

- [Some Possible Problems to Consider](#some-possible-problems-to-consider)
---

<br><br><br><br>

---

<h1 style="font-size: 2.5em;">PALLOC Config / Build / Install / Boot</h1>

<br>

# Setting up a QEMU+KVM VM
Already done: <br>
→ given 2 Cores, 8GB of RAM​, 120GB of SSD (60GB available for PALLOC, rest used by old EXTMEM). <br>
→ Ubuntu 22.04 minimal install​. <br>

My Ubuntu VM has already these kernels available for boot:
```
/boot/vmlinuz-6.8.0-59-generic   # stock kernel (default-ubuntu)
/boot/vmlinuz-6.8.0-60-generic   # stock kernel (default-ubuntu)
/boot/vmlinuz-5.15.0+            # patched kernel 5.15 with EXTMEM (old)
```

<br>

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

<br>

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
/boot/vmlinuz-6.8.0-59-generic		# stock kernel (default-ubuntu)
/boot/vmlinuz-6.8.0-60-generic		# stock kernel (default-ubuntu)
/boot/vmlinuz-5.15.0+			# patched kernel 5.15 with EXTMEM (old)
/boot/vmlinuz-6.6.0-palloc		# stock kernel 6.6
/boot/vmlinuz-6.6.0-palloc-dirty	# patched kernel 6.6 with PALLOC
```


---


<br><br><br><br>


<h1 style="font-size: 2.5em;">PALLOC Functionality / Experiment / Reproduce / Results</h1>

<br>

# Functionality
## Enable PALLOC
Verify that the palloc directory exists:
```
$ sudo ls /sys/kernel/debug/palloc
alloc_balance  control	debug_level  palloc_mask  use_mc_xor  use_palloc
```

Check if PALLOC is enabled:
```
$ sudo cat /sys/kernel/debug/palloc/use_palloc
0
```
0 means PALLOC is currently disabled.

Enable PALLOC and verify that it's active:
```
$ echo 1 | sudo tee /sys/kernel/debug/palloc/use_palloc
$ sudo cat /sys/kernel/debug/palloc/use_palloc
1
```


**SKIPPED THIS FOR NOW:**
## Detecting DRAM bank bits (for DRAM bank partitioning)
*For cache partitioning, just use the cache set bits instead of DRAM bank bits.*<br><br>
I'm inside a VM on a 3rd-gen Intel i7 (Ivy Bridge), so my physical memory mapping is likely "normal address bits", not XOR-mapped (like on Intel Haswell).

Select physical adddress bits to be used for page coloring (normal address bits):
```
$ echo 0x00183000 > /sys/kernel/debug/palloc/palloc_mask
```
--> select bit 12, 13, 19, 20. (total bins: 2^4 = 16)



**NEXT DID THIS:**
## Detecting cache bits mask (for cache partitioning)
Choose cache set bits.
Identify which physical address bits correspond to LLC cache sets.
For Ivy Bridge: typically bit 6–17 or similar, depending on cache size and associativity.
Example: if you want 16 “bins” in your cache, you could pick 4 bits:
Instead of DRAM bank bits, write the cache bits mask (palloc_mask).<br>
Example: selecting bits 12,13,14,15 for cache coloring:# Notes-Progress on Coloring support via PALLOC+patch usage
Guides used: <br>
https://github.com/heechul/palloc <br>
https://vatlidak-org.github.io/web/assets/md/setup_instructions/ <br>
https://docs.kernel.org/process/applying-patches.html <br>

PALLOC is a kernel-level memory allocator that exploits page-based virtual-to-physical memory translation to selectively allocate memory pages of each application to the desired DRAM banks. The goal of PALLOC is to control applications' memory locations in a way to minimize memory performance unpredictability in multicore systems by eliminating bank sharing among applications executing in parallel. PALLOC is a software based solution, which is fully compatible with existing COTS hardware platforms and transparent to applications (i.e., no need to modify application code).
PALLOC is a color-aware page-based physical memory allocator which replaces the buddy allocator for order-zero page allocations.

<br>

---

# Table of Contents

1. **PALLOC Config / Build / Install / Boot**
   - [Setting up a QEMU+KVM VM](#setting-up-a-qemukvm-vm)
   - [Kernel Compilation & Installation](#kernel-compilation--installation)
   - [Kernel Patch with PALLOC](#kernel-patch-with-palloc)

2. **PALLOC Functionality / Experiment / Reproduce / Results**
   - [Functionality](#functionality)
   - [Experiment](#experiment)
   - [Reproduce-Results](#reproduce-results)

3. **PALLOC Extension: finer_coloring.patch**
   - [Git Workflow Integration](#git-workflow-integration)
   - [Kernel-PALLOC patch with finer_coloring](#kernel-palloc-patch-with-finer_coloring)
   - [PALLOC + finer_coloring Functionality & Experiment](#palloc--finer_coloring-functionality--experiment)

4. **HugePages & Coloring coexistence**
   - [THP vs PALLOC: Global Behavior Possible Conflict](#thp-vs-palloc-global-behavior-possible-conflict)
   - [HugePages alongside Coloring: Functionality & Experiment](#hugepages-alongside-coloring-functionality--experiment)

- [Some Possible Problems to Consider](#some-possible-problems-to-consider)
---

<br><br><br><br>

---

<h1 style="font-size: 2.5em;">PALLOC Config / Build / Install / Boot</h1>

<br>

# Setting up a QEMU+KVM VM
Already done: <br>
→ given 2 Cores, 8GB of RAM​, 120GB of SSD (60GB available for PALLOC, rest used by old EXTMEM). <br>
→ Ubuntu 22.04 minimal install​. <br>

My Ubuntu VM has already these kernels available for boot:
```
/boot/vmlinuz-6.8.0-59-generic   # stock kernel (default-ubuntu)
/boot/vmlinuz-6.8.0-60-generic   # stock kernel (default-ubuntu)
/boot/vmlinuz-5.15.0+            # patched kernel 5.15 with EXTMEM (old)
```

<br>

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

<br>

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
/boot/vmlinuz-6.8.0-59-generic		# stock kernel (default-ubuntu)
/boot/vmlinuz-6.8.0-60-generic		# stock kernel (default-ubuntu)
/boot/vmlinuz-5.15.0+			# patched kernel 5.15 with EXTMEM (old)
/boot/vmlinuz-6.6.0-palloc		# stock kernel 6.6
/boot/vmlinuz-6.6.0-palloc-dirty	# patched kernel 6.6 with PALLOC
```


---


<br><br><br><br>


<h1 style="font-size: 2.5em;">PALLOC Functionality / Experiment / Reproduce / Results</h1>

<br>

# Functionality
## Enable PALLOC
Verify that the palloc directory exists:
```
$ sudo ls /sys/kernel/debug/palloc
alloc_balance  control	debug_level  palloc_mask  use_mc_xor  use_palloc
```

Check if PALLOC is enabled:
```
$ sudo cat /sys/kernel/debug/palloc/use_palloc
0
```
0 means PALLOC is currently disabled.

Enable PALLOC and verify that it's active:
```
$ echo 1 | sudo tee /sys/kernel/debug/palloc/use_palloc
$ sudo cat /sys/kernel/debug/palloc/use_palloc
1
```


**SKIPPED THIS FOR NOW:**
## Detecting DRAM bank bits (for DRAM bank partitioning)
*For cache partitioning, just use the cache set bits instead of DRAM bank bits.*<br><br>
I'm inside a VM on a 3rd-gen Intel i7 (Ivy Bridge), so my physical memory mapping is likely "normal address bits", not XOR-mapped (like on Intel Haswell).

Select physical adddress bits to be used for page coloring (normal address bits):
```
$ echo 0x00183000 > /sys/kernel/debug/palloc/palloc_mask
```
--> select bit 12, 13, 19, 20. (total bins: 2^4 = 16)



**NEXT DID THIS:**
## Detecting cache bits mask (for cache partitioning)
Choose cache set bits.
Identify which physical address bits correspond to LLC cache sets.
For Ivy Bridge: typically bit 6–17 or similar, depending on cache size and associativity.
Example: if you want 16 “bins” in your cache, you could pick 4 bits:
Instead of DRAM bank bits, write the cache bits mask (palloc_mask).<br>
Example: selecting bits 12,13,14,15 for cache coloring:
```
$ echo 0x0000f000 | sudo tee /sys/kernel/debug/palloc/palloc_mask
```
--> 0x0000f000 has bits 12,13,14,15 set to 1.

## CGROUP partition setting
Assign bins to a CGROUP so that all processes in that CGROUP allocate memory from specific bins: <br>
Enable palloc controller for child cgroups:
```
$ echo "+palloc" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
```
Create partition directory:
```
$ sudo mkdir /sys/fs/cgroup/part1
```
Assign bins 0-3 to part1 cgroup:
```
$ echo 0-3 | sudo tee /sys/fs/cgroup/part1/palloc.bins
```
Move your shell to this cgroup:
```
$ echo $$ | sudo tee /sys/fs/cgroup/part1/cgroup.procs
```
Enable PALLOC globally:
```
$ echo 1 | sudo tee /sys/kernel/debug/palloc/use_palloc

```
--> bin 0,1,2,3 are assigned to part1 CGROUP. <br>
--> from now on, all processes invoked from the shell use pages from the bins 0,1,2,3 only.

## Verify PALLOC enabled and everything worked correctly
Enable PALLOC and verify that it's active (otherwise the default buddy allocator will be used):
```
$ cat /sys/kernel/debug/palloc/use_palloc
$ cat /sys/fs/cgroup/part1/palloc.bins
$ cat /sys/fs/cgroup/part1/cgroup.procs
```

## Other options
Enable debug messsages visible through /sys/kernel/debug/tracing/trace. [Recommended]:
```
$ echo 1 | sudo tee /sys/kernel/debug/palloc/debug_level
```
Wait until at least 4 different colors are in the color cache. [Recommended]:
```
$ echo 4 | sudo tee /sys/kernel/debug/palloc/alloc_balance
```

## Disable support for transparent huge pages from kernel
Palloc doesn't work with transparent huge page. please disable this:
```
$ echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```


<br><br>


# Experiment
Currently, only processes explicitly added to /sys/fs/cgroup/part1/ use PALLOC with bins 0-3. <br>
How to add processes: <br>
- From your configured shell: Any command you run automatically uses PALLOC (since you moved your shell to part1)
- Any other process: Manually move it by writing its PID: `echo <PID> | sudo tee /sys/fs/cgroup/part1/cgroup.procs`
- New shell/terminal: Move it first: echo $$ | sudo tee /sys/fs/cgroup/part1/cgroup.procs, then all commands from that shell use PALLOC
<br>
All other processes (system services, GUI apps, other terminals) still use the default memory allocator (buddy) unless you explicitly move them to part1 or create additional partitions (part2, part3, etc.) with their own bin assignments.

## Testing PALLOC with microbenchmark.c
Ensure the current shell is moved into part1 cgroup:
```
$ echo $$ | sudo tee /sys/fs/cgroup/part1/cgroup.procs
```
Run the microbenchmark from current shell (in background), to allocate memory pages:
```
$ ./microbenchmark 4 &
```
Trace for palloc-related activity from the microbenchmark process:
```
$ sudo cat /sys/kernel/debug/tracing/trace | grep palloc | grep microbenchmark
```

The trace output shows PALLOC successfully allocating colored pages across different bins (0-3):
```
  ...
  microbenchmark-5962    [000] d..1. 89458.767783: palloc_find_cmap.constprop.0: 
  Found colored page pfn 2143507 color 3 seed 89459110159927 found/want 4/4
  microbenchmark-5962    [000] d..1. 89458.767842: palloc_find_cmap.constprop.0: 
  Found colored page pfn 2030272 color 0 seed 89459110219776 found/want 4/4
  microbenchmark-5962    [000] d..1. 89458.767905: palloc_find_cmap.constprop.0: 
  Found colored page pfn 2169921 color 1 seed 89459110282605 found/want 4/4
  microbenchmark-5962    [000] d..1. 89458.767966: palloc_find_cmap.constprop.0: 
  Found colored page pfn 2127138 color 2 seed 89459110341198 found/want 4/4
  ...
```


<br><br>


# Reproduce-Results
The aim of this section is to run extra experiments and to reproduce the paper's benchmarks-results (TBD). <br>
...
<br>


---


# Some Possible Problems to consider
- **Hardware/VM:** Running under QEMU/KVM, PALLOC has no real DRAM/cache control
- **palloc_mask:** Arbitrary (not derived from real address mapping)
- **Verification/Benchmarking:** Only functional tracing, no physical/latency validation (VM problem)
- **cgroups:** v1 vs v2

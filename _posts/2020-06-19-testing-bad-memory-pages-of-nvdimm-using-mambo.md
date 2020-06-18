# Validating bad memory pages of a PMEM device using Mambo simulator on POWER
  
## Introduction:

Error Injection is a key capability when it comes to simluating an flaw in a given enviroinment. It could be a DRAM, IO card or any piece of hardware which supports such injection. It is usually performed to verify whether kernel or the driver associated handles the injection properly and it possible recover from it. This blog is one such example which explains on how to validate the kernel handling of bad memory pages of a PMEM device.

Considering PMEM being a fairly new technology to be [implemented on POWER](https://harish-24.github.io/2020/03/15/persistent-memory-on-power-systems-and-its-validation.html), it supports all basic functionalities but does lack few functions like passphrase security, dimm health check etc. It is important here to note that if does not support error injection from neither hardware nor software.

Having said that, the kernel recovery/handling for a bad memory page of a PMEM device is already accepted upstream. How could that be tested by a developer if there is no hardware or a software support to induce an error? Testing this becomes challenging without such support. That is where simulation environment such as Mambo helps. The link [here](ftp://public.dhe.ibm.com/software/server/powerfuncsim/p9/packages) provides systemsim(mambo) rpm for power9 and with the binary for power9 under `/opt/ibm/systemsim-p9/run/p9/`.

## Environment Setup

To setup and run a mambo environment, the following needs to be exported.

* systemsim-p9(mambo) binary

* skiboot-lid

* skiboot.tcl

* vmlinux

* Base Image

* Pmem Image

* mambo_utils.tcl

Each of the above can be obtained in the following way

1. `skiboot-lid` can be built from the [skiboot source](https://github.com/open-power/skiboot.git)

2. `skiboot.tcl` is available in `external/mambo/` of the skiboot source.

3. The `vmlinux` can be built with any architecture here from kernel source. Make sure the built vmlinux contains the config options and support enabled for a PMEM device.

4. Base image is a disk formatted with root filesystem

5. Pmem image is a dummy disk created with `fallaocate` or `dd`. Make sure a filesystem is formatted to inject an error into its superblock.

6. `mambo_utils.tcl` is also part of skiboot source under `external/mambo/`

## Running with a PMEM device

We'll now see how each of this can be used to run a simulated mambo environment.

Create a file `run_mambo.sh` with the following contents:

```bash
export MAMBO_PATH=/opt/ibm/systemsim-p9/
export LINUX_CMDLINE="inst.sshd root=/dev/mtdblock0 rw console=hvc0"
export SKIBOOT=/home/skiboot/skiboot.lid
export SKIBOOT_ZIMAGE=/home/vmlinux
export PMEM_DISK=/home/pmem.img
export ROOTDISK=/home/disk.img
export SKIBOOT_AUTORUN=1
echo "Running with following env vars"
env|grep MAMBO
pushd $MAMBO_DIR
$MAMBO_PATH/run/p9/run_cmdline -f skiboot.tcl
popd
```

Note that the `LINUX_CMDLINE` contains a `root` parameter pointing to `/dev/mtdblock0` which indicates root filesystem location on booting. Optionally you can connect to a hvc console too.

Once all the required files are in place, we can trigger `./run_mambo.sh`.

The boot log looks like the following
```
1151748: (1151748): [    0.000154644,5] OPAL 47c599b9 starting...
1151748: (1151748): [    0.000159307,7] initial console log level: memory 7, driver 5
1151748: (1151748): [    0.000164864,6] CPU: P9 generation processor (max 4 threads/core)
1151748: (1151748): [    0.000170267,7] CPU: Boot CPU PIR is 0x0000 PVR is 0x004e1203
1151748: (1151748): [    0.000176972,7] OPAL table: 0x3010a630 .. 0x3010aba0, branch table: 0x30002000
1151748: (1151748): [    0.000184987,7] Assigning physical memory map table for nimbus
1151748: (1151748): [    0.000191031,7] FDT: Parsing fdt @0x1f00000
1151748: (1151748): [    0.001147671,5] Enabling Mambo console
1156229: (1156229): [    0.001151966,5] CHIP: Detected Mambo simulator
1234326: (1234326): [    0.001228385,5] CHIP: Chip ID 0000 type: P9N DD2.30
1485626: (1485626): [    0.001481039,5] PLAT: Detected Mambo platform
1706710: (1706710): [    0.001701972,5] CPU: All 1 processors called in...
1753198: (1753198): [    0.001748935,3] SBE: Master chip ID not found.
1815267: (1815267): [    0.001810111,5] mambo: Found bogus disk size: 0x19000000
1836799: (1836799): [    0.001831964,4] FLASH: No ffs info; using raw device only
1857836: (1857836): [    0.001853677,3] FLASH: Can't open ffs handle
1869025: (1869025): [    0.001864866,3] FLASH: Can't open ffs handle
1880214: (1880214): [    0.001876055,3] FLASH: Can't open ffs handle
1891403: (1891403): [    0.001887244,3] FLASH: Can't open ffs handle
1902592: (1902592): [    0.001898433,3] FLASH: Can't open ffs handle
1913781: (1913781): [    0.001909622,3] FLASH: Can't open ffs handle
WARNING: 1918333: (1918333): BogusDisk: Info: Trying using bogus disk #1 without TCL init
2031449: (2031449): [    0.002025737,3] NVRAM: Partition at offset 0x0 has incorrect 0 length
2036976: (2036976): [    0.002031740,3] NVRAM: Re-initializing (size: 0x00040000)
2197850: (2197850): [    0.002193587,5] STB: secure boot not supported
2206023: (2206023): [    0.002201708,5] STB: trusted boot not supported
2343186: (2343186): [    0.002337418,4] FLASH: Can't load resource id:0. No system flash found
2361096: (2361096): [    0.002355318,4] FLASH: Can't load resource id:1. No system flash found
2718972: (2718972): [    0.002714137,5] PCI: Resetting PHBs and training links...
2728159: (2728159): [    0.002724364,5] PCI: Probing slots...
2736882: (2736882): [    0.002733555,5] PCI Summary:
6740892: (6740892): [    0.006736785,5] INIT: Waiting for kernel...
6746935: (6746935): [    0.006742048,5] INIT: platform wait for kernel load failed
6763288: (6763288): [    0.006758869,5] INIT: 64-bit LE kernel discovered
6776853: (6776853): [    0.006770097,3] STB: container NOT VERIFIED, resource_id=0 secureboot not yet initialized
6783175: (6783175): [    0.006777924,3] OCC: Unassigned OCC Common Area. No sensors found
15660206: (15660206): [    0.015652933,5] INIT: Starting kernel at 0x20010000, fdt at 0x3054d230 15838 bytes
...
...
...
```

Make sure the following is observed while booting

```
451895456: (451895429): [    0.623866] nd_pmem namespace0.0: unable to guarantee persistence of writes
452009056: (452009029): [    0.624088] radix-mmu: Mapped 0xc000020000000000-0xc000020040000000 with 1.00 GiB pages
```

This indicates pmem device is attached as a result of exporting `PMEM_DISK` variable.
Please also find the virtual address range `0xc000020000000000-0xc000020040000000` of the pmem device eventually will be used for injecting an error.

Login to the kernel and check pmem device existence

```
# ls /dev/pmem0
/dev/pmem0
```

Once the device is available, we will need to drop out of the console to reach the mambo systemsim console
Here we use the `inject_mce_ue_on_addr` function to inject an error to the superblock of the pmem device from the mambo console. 

```bash
systemsim % source /home/skiboot/external/mambo/mambo_utils.tcl
systemsim % inject_mce_ue_on_addr 0xc000020000001165
systemsim % c
```

Executing `c` returns back to the booted kernel. Then we try to mount the error injected device and verify the kernel handling the badsuperblock of the pmem device.

```bash
# mount /dev/pmem0 /mnt
mount: mounting /dev/pmem0 on /mnt/ failed: Invalid argument
#
# dmesg | grep -i pmem
[1040551.890187] EXT4-fs (pmem0): Can't read superblock on 2nd try
```

This [link](https://lkml.org/lkml/2019/8/5/59) provides the patchset which handles the failure.

Thanks for reading this blog on validating handling of bad memory pages of a pmem device by the kernel!

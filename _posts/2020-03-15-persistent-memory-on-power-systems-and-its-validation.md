# Persistent memory on POWER systems and its validation

## Introduction:
This is a blog on how validation of persistent memory/storage class memory(SCM) has evolved over a period of time on POWER architecture. Validating a SCM typically needs a persistence memory device to operate on. But more often than not devices are not production-ready and gets delayed due to various external factors. It becomes probable for both development and test to have a simulation to get together the software pieces required before the harware actually arrives.

The following config options are required for PMEM support

```
CONFIG_ZONE_DEVICE=y
CONFIG_MEMORY_HOTPLUG=y
CONFIG_MEMORY_HOTREMOVE=y
CONFIG_TRANSPARENT_HUGEPAGE=y
CONFIG_ACPI_NFIT=m
CONFIG_X86_PMEM_LEGACY=m
CONFIG_OF_PMEM=m
CONFIG_LIBNVDIMM=m
CONFIG_BLK_DEV_PMEM=m
CONFIG_BTT=y
CONFIG_NVDIMM_PFN=y
CONFIG_NVDIMM_DAX=y
CONFIG_FS_DAX=y
CONFIG_DAX=y
CONFIG_DEV_DAX=m
CONFIG_DEV_DAX_PMEM=m
CONFIG_DEV_DAX_KMEM=m
```

In x86 world, the kernel had support to emulate primary memory of the system to be treated as Persistent memory through a boot-time parameter `memmap`
With enabling few config options required for SCM, this was possible.

Hence, the first piece to start with was qemu which had needed functionalities. With the help of few pathes to qemu which introduced `papr_scm` module, this was achieved. Introduction of a new memory model "nvdimm", enabled to get a new custom qemu binary with fake SCM support.

```
<memory model='nvdimm' access='shared'>
<source>
<path>/tmp/nvdimm</path>
</source>
<target>
<size unit='KiB'>1048576</size>
<label>
<size unit='KiB'>4096</size>
</label>
</target>
<address type='dimm' slot='0'/>
</memory>
```

This allowed to uncover early defects on a new architecture.

Meanwhile, due to further delay in hardware parallel work was in progress provide a DRAM as persistent memory on a para-virtualized hardware such as pHYP based PowerVM. This was eventually called virtual PMEM(vPMEM). It had its own advantage and disadvantage due to obvious reasons. Thorugh hypervisor and firmware patches, it was made persistent across a logical partition reboot on para virtualized environment.

The system resources are split into memory pools to be used as PMEM regions at OS level. This pooled regions are allocated to logical partitions according to the need. That is eventually passed on to OS level through dynamic device tree entries at boot time. These can be viewed at the OS level by the userspace utility called `ndctl`

```bash
# ndctl list -R
[
  {
    "dev":"region1",
    "size":67108864000,
    "available_size":67108864000,
    "max_available_extent":67108864000,
    "type":"pmem",
    "iset_id":877526644077441543,
    "persistence_domain":"unknown"
  },
  {
    "dev":"region0",
    "size":53687091200,
    "available_size":53687091200,
    "max_available_extent":53687091200,
    "type":"pmem",
    "iset_id":8243633388223968796,
    "persistence_domain":"unknown"
  }
]
```

Quite a number of new test scenarios were introduced at OS level to test the stack. In a couple of months time, a lot of bug fixes and new feature implementations we were ready to roll out. Once kernel was ready, we started to use a NVDIMM-N card on a OPAL based PowerNV environment. Though the kernel code base is same to handle pmem devices, a new driver had to be enabled to talk to hypervisor on a OPAL environment. This driver was named `of_pmem` owing to open firmware on OPAL systems. NVDIMM-N was a legacy harware and supports single namespace per region. This was handled at a test case level.

The following tests were enabled upstream to uncover defects. These automated tests can be found [here](https://github.com/avocado-framework-tests/avocado-misc-tests/). These are tests based on [avocado framework](https://github.com/avocado-framework/avocado/).
You can choose to take a look at [PMEM library](https://github.com/avocado-framework/avocado/blob/master/avocado/utils/pmem.py) providing basic command ndctl functionality. Using this API, a set of tests these test cases cover

* All supported ndctl options including
  * Single and multiple, at region and namespace level
    1. create
    2. destroy
    3. enable
    4. disable
  * Supported modes (raw, sector(BTT), fsdax and devdax)
  * Supported memory maps (only for fsdax and devdax)
  * Numa associativity

* xfstest test using pmem devices as block devices to find any filesystem specific issues on fsdax/sector modes.

* FIO stressors

* HTX stressors

* Few selftest from [ndctl source](https://github.com/pmem/ndctl/)


To setup and run these tests using avocado framework, [tests](https://github.com/open-power-host-os/tests/) repository can be leveraged here. This installs latest avocado on the host and runs the provided test config.

```
1. git clone https://github.com/open-power-host-os/tests/
2. cd tests
3. create a config with following content config/tests/host/nvdimm.cfg
avocado-misc-tests/memory/ndctl_selftest.py
avocado-misc-tests/memory/ndctl.py avocado-misc-tests/memory/ndctl.py.data/ndctl.yaml "--mux-filter-out /run/config/mode_types"
avocado-misc-tests/fs/xfstests.py avocado-misc-tests/fs/xfstests.py.data/nvdimm.yaml
avocado-misc-tests/fs/xfstests.py avocado-misc-tests/fs/xfstests.py.data/nvdimm_log.yaml
avocado-misc-tests/io/disk/fiotest.py avocado-misc-tests/io/disk/fiotest.py.data/fio-pmem.yaml
avocado-misc-tests/generic/htx_test.py avocado-misc-tests/generic/htx_test.py.data/htx_pmem.yaml "--execution-order tests-per-variant"

4. python avocado-setup.py --install-deps
5. python avocado-setup.py --run-suite host_nvdimm
```

Note: HTX tests have dependencies on custom rpms and hence cannot be included without a valid rpm.

Thanks for reading this blog!! Hope you enjoyed it!!

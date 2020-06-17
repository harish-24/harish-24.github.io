# Device tree interpretation of a pseries guest memory

## Introcudtion:

This blog explains on how memory on a pseries guest is tracked from device-tree perspective. A pseries guest can be of different types, KVM or pHYP based PowerVM both using virtualization.

In case of KVM, we all know the possibility of memory hotplug and unplug. This is adding a memory device to a guest when it is in use. But there are limitations as to how much memory can be added to the guest defined by a libvirt XML property <maxMemory>. When this property is defined, the guest device-tree is populated with a dynamic memory reconfiguration.

```
/proc/device-tree/ibm,dynamic-reconfiguration-memory/ibm,dynamic-memory
/proc/device-tree/ibm,dynamic-reconfiguration-memory/ibm,dynamic-memory-v2
```

### v2 Representation:

v2 is an alternate and newer way of dynamic reconfigurable memory.

which provides information on the number of LMB (logical memory block) as per the maxMemory size. The lmb-size of the guest is defined in ibm,lmb-size property `/proc/device-tree/ibm,dynamic-reconfiguration-memory/ibm,lmb-size`

Note: All the fields are represented in hex.

```
# lsprop /proc/device-tree/ibm,dynamic-reconfiguration-memory/ibm,lmb-size
		 00000000 10000000
```

We have a 256MB LMB size here.

A sample output of the dynamic memory:
```
# lsprop /proc/device-tree/ibm,dynamic-reconfiguration-memory/ibm,dynamic-memory
		 00000020 00000000 00000000 00000000
		 00000000 ffffffff 000000a0 00000000
		 10000000 00000000 00000000 ffffffff
		 000000a0 00000000 20000000 00000000
		 00000000 ffffffff 000000a0
		 ...
		 ...
```

Let us see what each field represents.
* `00000020` - The first field represents the N number of LMBs, which is 32 in decimal. Considering we have a LMB size of 256MB it adds up to 8GB of maxMemory.
The first field being 32, there are 32 arrays representing the LMBs. Each array contains 6 fields.

Let us take the next 6 fields here

* `00000000 00000000` - Logical address of the start of the LMB encoded as a 64bit integer

* `00000000` - DRC index

* `00000000` - Four bytes reserved for expansion

* `ffffffff` - Associativity index

* `000000a0` - A 32bit flags word

Now, let us take a look at the newer implementation of dynamic memory

```
# lsprop /proc/device-tree/ibm,dynamic-reconfiguration-memory/ibm,dynamic-memory-v2
		 00000001 0000018e 00000000 20000000
		 80000002 00000001 00000008
```

* `00000001` - Number of LMB sets

Fields of each set

* `0000018e` - Number of sequential LMBs in the entry represented by a 32bit integer

* `00000000 20000000` - Logical address of the first LMB in the set encoded as a 64bit integer 

* `80000002` - DRC index

* `00000001` - Associativity index

* `00000008` - A 32bit flags word that applies to all the LMBs in the set

This translates to

```
nr-lmbs(size): 	398(99.5G)
addr: 		0000000020000000
drc: 		80000002
a_index: 	00000001
flags: 		00000008
```

Note: Associativity index is used as an index into ibm,associativity-lookup-arrays, which derives at the numa node the LMB belong to. Let us see how.

```
# lsprop /proc/device-tree/ibm,dynamic-reconfiguration-memory/ibm,associativity-lookup-arrays 
		 00000002 00000004 00000000 00000000 00000000
		 00000000 00000000 00000000 00000002 00000002
```

In v2, the associativity-lookup array contains two nodes with four entries each.
`Index 0:` 00000000 00000000 00000000 00000000
`Index 1:` 00000000 00000000 00000002 00000002

With the help of `/proc/device-tree/rtas/ibm,associativity-reference-points` and the associativity index, the node the LMB set is assigned to can be arrived at.

```
# lsprop /proc/device-tree/rtas/ibm,associativity-reference-points
		 00000004 00000002
```

Representing lookup array index being 1 `(a_index: 00000001)` and `00000004` index from reference point, the node number can be mapped as "node 2".

```
# lsprop /proc/device-tree/rtas/ibm,associativity-reference-points
[1][4]:   00000000 00000000 00000002 00000002
	        	   	     ^^^^^^^^
```

Hope you enjoyed reading this!

### Reference:
* [ppc-spapr-hotplug.txt](https://github.com/qemu/qemu/blob/master/docs/specs/ppc-spapr-hotplug.txt)

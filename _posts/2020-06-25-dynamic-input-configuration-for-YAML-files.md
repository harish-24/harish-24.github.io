# Dynamic input configuration for YAML files in Avocado


## Introduction

Functional verification of Hosts has been quite straight forward when the environment is static. But when the tests has to cover a majority of test scenarios, reusability becomes the key. The test scripts has to be more robust and generic to handle/cover what the environment requires. In such cases, it is important to proactively understand the need of inputs before designing a test script. In a automated environment like Continuous intergration or Regression, these inputs must be fed dynamically to the test environment. This blog talks about how Dynamic input configuration helps tests cover various scenarios/tests running through avocado and eventually make us achieve the end goal of CI/CR.

## YAML Files for tests

For those who are not aware, [Avocado](https://avocado-framework.readthedocs.io/) is a frameowrk that is widely used by companies like Redhat, IBM for Linux kernel validation. It also provides various features and enhancements for guest specific validation through [avocado-vt](https://github.com/avocado-framework/avocado-vt/). The repository [avocado-misc-tests](https://github.com/avocado-framework-tests/avocado-misc-tests/) maintains tests for host validation which leverages APIs from avocado framework.

Over and above, we have a test wrapper framework which provides assistance in installing and running of tests as suites. The details of the [tests](https://github.com/open-power-host-os/tests) framework and how one can leverage it use to run as suites can found [here](https://narasimhan-v.github.io/2020/06/12/Writing-Avocado-Test-Suites.html).

If you take a look at the test cases, most often than not it has a YAML file associated with it for specific inputs. Those inputs has to be runtime configurable in a environment like test bucket coverage or CI/CR. The tests wrapper framework already has a mechanism to replace input values of the given input config file matching yaml files with `--input-file` option. But it has its own limitations. They are listed as follows:

1. The key in the  input config, matching any yaml file will be replaced

2. There is no way to specify different values to same key at different branches

3. Lacks branch level control

All these are addressed with newer implementation [here](https://github.com/open-power-host-os/tests/pull/224/commits/687c9c6912990cfe443b290ee9974a442f391f0e).

Now we can see how this feature can be leveraged with an example.

Lets take [xfstests.yaml](https://github.com/avocado-framework-tests/avocado-misc-tests/blob/master/fs/xfstests.py.data/xfstests.yaml) as file here. A portion of it is used here for the explanation to be simple.

```bash
$ cat fstests.cfg
avocado-misc-tests/fs/xfstests.py avocado-misc-testsfs/xfstests.py.data/xfstests.yaml
avocado-misc-tests/fs/fsx.py

$ cat xfstests.yaml
setup:
    skip_dangerous: True
    scratch_mnt: '/mnt/scratch'
    fs_type: !mux
        fs_ext4:
            fs: 'ext4'
            exclude: '2,4-7,203'
            gen_exclude: '1-10,30-45'
            share_exclude: '1-2'
        fs_xfs:
            fs: 'xfs'
            # Exclude only if test_range not provided
            exclude: '2,4-7,203'
            gen_exclude: '1-10,30-45'
            share_exclude: '1-2'
```

With the mentioned patch we can provide inputs leaf nodes with different parent branch

input.txt
```
[fstests]
setup.skip_dangerous=False
setup.fs_type.fs_ext4.exclude=2-15,304,205
setup.fs_type.fs_xfs.exclude=1-10
setup.fs_type.fs_xfs.gen_exclude=1,2
```
Note that cfg has a section name `fstests` matching the suite `fstests.cfg`

When run with the above input file using `--input-file`, this makes sure that the yaml looks like

```cfg
# xfstests.yaml
setup:
    skip_dangerous: false
    scratch_mnt: /mnt/scratch
    fs_type: !mux
        fs_ext4:
            fs: ext4
            exclude: 2-15,304,205
            gen_exclude: 1-10,30-45
            share_exclude: 1-2
        fs_xfs:
            fs: xfs
            exclude: 1-10
            gen_exclude: 1-10,30-45
            share_exclude: 1,2
```
Thus eradicating the limitations specified above.


## Using * Notation

To enhance it further [this patch](https://github.com/open-power-host-os/tests/pull/224/commits/e022628d7ee02d916f36274e792a15a3f425b114) we also have an option to provide inputs matching multiple yaml files. This can be achieved through a regex like input through special character `*`.

Lets see another example to explain this. In case `share_exclude` has to be updated for both the filesystem types, the following config would be able to achieve this

```
[fstests]
setup.skip_dangerous=False
setup.fs_type.fs_ext4.exclude=2-15,304,205
setup.fs_type.*.share_exclude=1,250
```

which translates to
```
# xfstests.yaml
setup:
    skip_dangerous: false
    scratch_mnt: /mnt/scratch
    fs_type: !mux
        fs_ext4:
            fs: ext4
            exclude: 2-15,304,205
            gen_exclude: 1-10,30-45
            share_exclude: 1,250 -----------> Updated here
        fs_xfs:
            fs: xfs
            exclude: 1-10
            gen_exclude: 1-10,30-45
            share_exclude: 1,250 -----------> Here as well
```

The same notation can be used at any level of input except for the leaf level as that is the key in the yaml file to be updated.

## Implementation

As for the implementation of this, it would have been straight forward had there been no `!mux` decorator in the yaml files. This is a necessity for Avocado framework as the test scenarios are multiplexed based on the placement of the decorator in yamls file.

```
# avocado variants -m xfstests.yaml 
Multiplex variants (2):
Variant fs_ext4-disk-bc2b:    /run/setup/loop_type/disk, /run/setup/fs_type/fs_ext4
Variant fs_xfs-disk-e0b0:    /run/setup/loop_type/disk, /run/setup/fs_type/fs_xfs
```

Without the decorator, achieving leaf level would be a cake walk especially with the [PyYaml](https://pyyaml.org/wiki/PyYAMLDocumentation) module. Just load the yaml as dictionary, replace the values in dictionary and dump it back.

With the `!mux` decorator in place, a number of overrides becomes essential here. `yaml` object needs to be wrapped with a Loader object which inturn uses a `Constructor` handling the decorator so that yaml safely loads file contents to a dictionary. The `yaml.dump()` optinally takes a `Dumper` argument which can be enabled to dump the decorator at the right place through what is called `Representer`. This is to store/wrap the tree below the decorator and store it as is on the dictionary and restore back with the decorator. A `OverrideConstructor` wraps the contents of the tree and a `OverrideDumper` helps dumping back the data to the file with the decorator. This [link](https://github.com/open-power-host-os/tests/pull/224/files#diff-c7c73da3c9a0a695d49f36e0a69e713f) shows how the data is being wrapped and dumped with the Dumper.

```python
class OverrideConstructor(yaml.constructor.SafeConstructor):
    def __init__(self):
        yaml.constructor.SafeConstructor.__init__(self)

    def construct_undefined(self, node):
        data = getattr(self, 'construct_' + node.id)(node)
        datatype = type(data)
        wraptype = type('TagWrap_'+datatype.__name__, (datatype,), {})
        wrapdata = wraptype(data)
        wrapdata.tag = lambda: None
        wrapdata.datatype = lambda: None
        setattr(wrapdata, "wrapTag", node.tag)
        setattr(wrapdata, "wrapType", datatype)
        return wrapdata

class OverrideLoader(OverrideConstructor, yaml.loader.SafeLoader):

    def __init__(self, stream):
        OverrideConstructor.__init__(self)
        yaml.loader.SafeLoader.__init__(self, stream)

class OverrideRepresenter(yaml.representer.SafeRepresenter):
    def represent_data(self, wrapdata):
        tag = False
        if type(wrapdata).__name__.startswith('TagWrap_'):
            datatype = getattr(wrapdata, "wrapType")
            tag = getattr(wrapdata, "wrapTag")
            data = datatype(wrapdata)
        else:
            data = wrapdata
        node = super(OverrideRepresenter, self).represent_data(data)
        if tag:
            node.tag = tag
        return node

class OverrideDumper(OverrideRepresenter, yaml.dumper.SafeDumper):
    def __init__(self, stream, default_style=None, default_flow_style=False,
		 ...)
        OverrideRepresenter.__init__(self, default_style=default_style,
                                     default_flow_style=default_flow_style,
                                     sort_keys=sort_keys)
```

Note that the Constructor wrap the datatype of its tree as well to restore back properly.

Thank you for reading through and hope you have enjoyed what this blog explains!!

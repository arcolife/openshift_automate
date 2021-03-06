### Add RHEV based VM

#### Prerequisites for usage

- The client from where scripts are supposed to be run, must be able to access RHEV api interface
- On Fedora it is necessary to install `ovirt-engine-sdk-python` python library. This can achieved with below command


```
# dnf install ovirt-engine-sdk-python
```

- If not executed directly from RHEV manager, ensure to pass correct location for `--rhevcafile` (How to get rhevcafile check with RHEV system administrator)
By default `--rhevcafile` will be searched and tried to access at `/etc/pki/ovirt-engine/ca.pem` which is at that location on RHEV manager, for other cases
it is necessary to have access to RHEV `ca.pem` certificate

#### Properties

Using `add_vm_rhev.py` it is possible to create desired number of virtual machines from particular template.VM disk can be `thin` - what is default in RHEV, or `preallocated` if `vmdiskpreallocated=yes` is specified when script
is executed.
For most cases using `thin` volume disk type for virtual macihine VM is satisfactory, however for cases where is necessary to
have virtual machines with minimum possible latency ( ideal candidates for this  are OpenShift masters and/or ETCD servers), then
it is  strogly advisable to have virtual machines with `preallocated` storage

For more information regarding RHEV virtual disk, refer to RHEV documentation chapter [Understanding Virtual Disks](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Virtualization/3.1/html/Administration_Guide/Understanding_virtual_disks.html)


If there is necessary to have virtual machine with multiple (additional) disks attached to it, then specifying
`--numdisks` with number of additional disks will do that. It is necessary to mention that specifying high number
for `numdisks` will eventually fail due to kernel limitation. Reasonable number for `numdisks` for most cases is
up to 40, what is going to attach 40 disk devices to virtual machine. This is limitation from RHEV virtual machine side, other limitation could appear
if size of allocated disks cannot be accomodated at storage side.

For case when is chosen to create `preallocated` storage type and/or many disk to be attached to
virtual machine with `numdisks` the time it takes to create virtual machine(s) can be long and it is
recommended to use these two options only when really necessary.

For more details, check `USAGE` section



#### USAGE

For details how to use [add_vm_rhev.py](https://github.com/ekuric/openshift/blob/master/_rhev/add_vm_rhev.py) script refer to below help output
```
# python add_vm_rhev.py -h
usage: add_vm_rhev.py [-h] --url URL --rhevusername RHEVUSERNAME
                      --rhevpassword RHEVPASSWORD [--rhevcafile RHEVCAFILE]
                      [--memory MEMORY] [--cluster CLUSTER]
                      [--vmtemplate VMTEMPLATE] [--nicname NICNAME]
                      [--num NUM] --vmprefix VMPREFIX [--disksize DISKSIZE]
                      [--vmcores VMCORES] [--vmsockets VMSOCKETS]
                      [--storagedomain STORAGEDOMAIN] [--network NETWORK]
                      [--addstorage ADDSTORAGE]
                      [--vmdiskpreallocated VMDISKPREALLOCATED]
                      [--diskpreallocated DISKPREALLOCATED]
                      [--numdisks NUMDISKS]

Script to create RHEV based virtual machines

optional arguments:
  -h, --help            show this help message and exit
  --url URL             RHEV_URL
  --rhevusername RHEVUSERNAME
                        RHEV username
  --rhevpassword RHEVPASSWORD
                        RHEV password
  --rhevcafile RHEVCAFILE
                        Path to RHEV ca file
  --memory MEMORY       Memory size for RHEV VM, eg 2 means virtual machine
                        will get 2GB of memory
  --cluster CLUSTER     RHEV Cluster - in which cluster create vm
  --vmtemplate VMTEMPLATE
                        RHEV template
  --nicname NICNAME     NIC name for vm
  --num NUM             how many virtual machines to create
  --vmprefix VMPREFIX   virtual machine name prefix
  --disksize DISKSIZE   disk size to attach to virtual machine in GB - passing
                        1 will create 1 GB disk and attach it to virtual
                        machine, default is 1 GB
  --vmcores VMCORES     How many cores VM machine will have - default 1
  --vmsockets VMSOCKETS
                        How many sockets VM machine will have - default 1
  --storagedomain STORAGEDOMAIN
                        which storage domain to use for space when allocating
                        storage for VM If not sure which one - check web
                        interface and/or contact RHEV admin
  --network NETWORK     Where to connect eth0 network interface network
                        specified here has to be present in RHEV environment
                        prior trying to create virtual machines, default is
                        ovirtmgmt network
  --addstorage ADDSTORAGE
                        wheather or not to attach additional storage from
                        storage domain to this VM
  --vmdiskpreallocated VMDISKPREALLOCATED
                        For new VM use preallocted disk instead of thin, by
                        default RHEV will use thin if preallocated is not
                        specified
  --diskpreallocated DISKPREALLOCATED
                        If there is additional disk added to VM this will
                        define will be disk be preallocated instead of default
                        thin
  --numdisks NUMDISKS   how many disks to attach to particular VM - be
                        reasonalbe, trying to attach too many disks will not
                        work due to kernel limits

```

It is assumed that only one core per socket hardware supports, in all examples below it is

```
--vmcores=1
```

if it necessary (and if hardware supports that) it is possible to add more cores per socket by adding desired value to `--vmcores` parameter

#### Example 1:

Create 10 machines with 8 GB of memory, 4  sockets and 1 core per sockets. Attach additional 10 GB disk to every created VM machine
```
# python add_vm_rhev.py --url="RHEV_API_WEB - eg https://rhv-m.local/ovirt-engine/api" --rhevusername="admin@internal" --rhevpassword="mypasswd" --memory=8  --vmtemplate="test-template" --disksize=10 --storagedomain=iSCSI --vmsockets=4 --vmprefix=openshift_master --num=10
```

In above example , below values

- url
- rhevusername
- rhevpassword
- storagedomain
- vmtemplate

are supposed to be known in advance, RHEV system administrator can provide them, or appropriate user with rights to create
and start virtual machine in RHEV environment

It is strongly advised to use proper value for `vmprefix` parameter

`vmprefix` parameter can help to differ machines from others

#### Example 2:

Create 3 machines with 16 GB of memory, 16 CPU sockets and 1 core per sockets. Attach 50 GB disk to newly created machines. Put virtual machines
disk on `preallocated` disk type

Tag all machines with `openshift_master` prefix

```
# python add_vm_rhev.py --url="RHEV_API_WEB - eg https://rhv-m.local/ovirt-engine/api" --vmdiskpreallocated=yes --rhevusername="admin@internal" --rhevpassword="mypasswd" --memory=16  --vmtemplate="test-template" --disksize=50 --storagedomain=iSCSI --vmsockets=16 --vmprefix=openshift_master --num=3
```
Creating virtual machines with `preallocated` disk can lead to longer VM creation time. In nutshell RHEV will do `dd` to disk which will be
used by virtual machine. The bigger disk specified in VM template, the longer creation time with `preallocated` disk will last


Create 3 machines without attaching additional storage to them

```
# python add_vm_rhev.py --url="RHEV_API_WEB - eg https://rhv-m.local/ovirt-engine/api" --rhevusername="admin@internal" --rhevpassword="mypasswd" --memory=16  --vmtemplate="test-template" --vmprefix=openshift_master --num=3 --addstorage=no
```

#### Example 3 - create amazon like machines - some examples

- m4.large : cores = 1 , sockets = 2 , memory = 8 GB
```
python add_vm_rhev.py --url="https://rhv-m.local/ovirt-engine/api" --rhevusername="admin@internal" --rhevpassword="mypasswd" --vmtemplate="test-template" --disksize=1 --storagedomain=iSCSI --vmprefix=elko_node   --num=1 --vmsockets=2  --memory=8

```
- m4.xlarge : cores = 1 , socket = 4 , memory = 16 GB
```
# python add_vm_rhev.py --url="https://rhv-m.local/ovirt-engine/api" --rhevusername="admin@internal" --rhevpassword="mypasswd" --vmtemplate="test-template" --disksize=1 --storagedomain=iSCSI --vmprefix=elko_node1   --num=1 --vmsockets=4 --memory=16
```

- m4.2xlarge : cores = 1, socket = 8, memory = 32 GB

```
python add_vm_rhev.py --url="https://rhv-m.local/ovirt-engine/api" --rhevusername="admin@internal" --rhevpassword="mypasswd" --vmtemplate="test-template" --disksize=1 --storagedomain=iSCSI --vmprefix=elko_node1   --num=1 --vmsockets=8 --memory=32
```

- m4.4xlarge : cores = 1, socket = 16, memory = 64 GB

```
python add_vm_rhev.py --url="https://rhv-m.local/ovirt-engine/api" --rhevusername="admin@internal" --rhevpassword="mypasswd" --vmtemplate="test-template" --disksize=1 --storagedomain=iSCSI --vmprefix=elko_node1   --num=1 --vmsockets=16 --memory=64
```

- For other cases, change `vmcores`, `vmsockets` and `memory` parameters to correspond desired VM size.

**Important:** pay attention on `--vmcores` value if changed from default value ( --vmcores=1 ) and ensure that underlaying hardware
is capable to run more cores than one per socket.


#### Cleaning up environment

After using virtual machines there is need to clean up them to free up resource in RHEV environment.
[delete_rhev_vm](https://github.com/ekuric/openshift/blob/master/_rhev/delete_rhev_vm.py) script can help with this
Refer below help output on how to use it

```
# python delete_rhev_vm.py -h
usage: delete_rhev_vm.py [-h] --url URL --rhevusername RHEVUSERNAME
                         --rhevpassword RHEVPASSWORD [--rhevcafile RHEVCAFILE]
                         [--vmprefix VMPREFIX] [--action ACTION]

Script to start RHEV boxes

optional arguments:
  -h, --help            show this help message and exit
  --url URL             RHEV_URL
  --rhevusername RHEVUSERNAME
                        RHEV username
  --rhevpassword RHEVPASSWORD
                        RHEV password
  --rhevcafile RHEVCAFILE
                        Path to RHEV ca file, default is /etc/pki/ovirt-
                        engine/ca.pem
  --vmprefix VMPREFIX   virtual machine name prefix, this prefix will be used
                        to select machines agaist selected action will be
                        executed
  --action ACTION       What action to execute. Action can be:start - start
                        machinesstop - stop machines collect - collect
                        ips/fqdn of machines
```

actions `delete_rhev_vm.py` can take are as showed below

- start - start VM based on virtual machines prefix
- stop - stop VM based on virtual machines prefix
- delete - delete VM based on virtual machines prefix
- collect - when passed option `collect` it will check virtual machines based on desired prefix (`vmprefix`) and collect
ip addresses and fqdn of these machines.

#### Example 1

Stop all virtual machines which names start with prefix "openshift_master"

```
# delete_rhev_vm.py --url="<rhev url api eg. https://rhv-m.local/ovirt-engine/api" --rhevusername="admin@internal" --rhevpassword="secret_pass" --vmprefix=openshift_master --action stop
```
#### Example 2

Start all virtual machines which name start with prefix `openshift_master`
```
# delete_rhev_vm.py --url="<rhev url api eg. https://rhv-m.local/ovirt-engine/api" --rhevusername="admin@internal" --rhevpassword="secret_pass" --vmprefix=openshift_master --action start
```

#### Example 3

Delete all virtual machines which name start with `openshift_master` from RHEV cluster.
This step is destructive, so be careful and pass correct `vmprefix` value. Machines which run will not be deleted, they first need to be stopped, already stopped machines
will be removed from RHEV cluster and cannot be recovered
```
# delete_rhev_vm.py --url="<rhev url api eg. https://rhv-m.local/ovirt-engine/api" --rhevusername="admin@internal" --rhevpassword="secret_pass" --vmprefix=openshift_master --action delete
```

#### Example 4
Collect `fqdn` and `ip` of all machines which name starts with `openshift_master`

```
# delete_rhev_vm.py --url="<rhev url api eg. https://rhv-m.local/ovirt-engine/api" --rhevusername="admin@internal" --rhevpassword="secret_pass" --vmprefix=openshift_master --action collect
```
In last example, if machine(s) are not started propelry in RHEV cluster and if they do not have assigned fqdn/ip then these values
will not be collected


#### Future steps - TODO
implement "amazon like" machine tagging following below
-  [amazon virtual cores](https://aws.amazon.com/ec2/virtualcores/)
- [amazon instance types](https://aws.amazon.com/ec2/instance-types/)

and instead adding `vmcores` and `vmsockets` use "amazon like" approach ( eg. m4.large, m4.xlarge, ... ) to create desired machine

# Setup a Ceph iSCSI GW

In this lab we assume you already completed Lab 1 and you have a working Ceph cluster

**WARNING:**
iSCSI gw support in CentOS7 is still **experimental** we will use an unofficial kernel and tools that are not the versions included in the distro. As a matter of fact, true load-balance MPIO is not supported, it only works in active/standy failover configurations

We will need the following **unofficial** packages installed in all iSCSI GW hosts:

  * kernel: a 4.16 kernel with needed patches (https://github.com/ceph/ceph-client)
  * ceph-iscsi-config: configuration modules for managing iscsi gateways for ceph (https://github.com/ceph/ceph-iscsi-config)
  * ceph-iscsi-cli: cli for managing ceph/iscsi gateways built using LIO (https://github.com/ceph/ceph-iscsi-cli)
  * tcmu-runner: userspace side of the LIO TCM-User backstore, compiled with last librbd1 with needed locking support (https://github.com/open-iscsi/tcmu-runner/)
  * python-rtslib: Python library for configuring the Linux kernel-based multiprotocol SCSI target (LIO) (https://github.com/open-iscsi/rtslib-fb)

To simplify deployment we have created a yum repository with the right version for all needed packages and it's dependencies (http://yumrepo.modularit.net/repos/ceph-iscsi-centos-7/ceph-iscsi-centos-7.repo)

## Manual setup

  * Install our additional repo:

```shell
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h "curl http://yumrepo.modularit.net/repos/ceph-iscsi-centos-7/ceph-iscsi-centos-7.repo -o /etc/yum.repos.d/ceph-iscsi-centos-7.repo"; done
```

  * Install kernel in gateway hosts

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h "yum -y install kernel"; done
```

  * Make sure the configured kernel for next boot is the new 4.16 kernel

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h "grub2-editenv list"; done
```

  * If it is not, configure grub to boot from first kernel (the one we just installed)

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h "grub2-set-default 0"; done
```

  * Reboot updated hosts

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h "reboot"; done
```

  * When hosts are back online, check booted kernel is correct version (4.x)

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h "uname -a"; done
```

  * Create rbd pool

```
ceph osd pool create rbd 64 64
ceph osd pool application enable rbd rbd
```

  * Install needed packages on gateway hosts

```
for h in ceph-mon{1,2,3}; do echo $h; ssh $h yum install tcmu-runner ceph-iscsi-cli ceph-iscsi-config ; done
```

  * Create gateway file (with secure=false)

```shell
cat > iscsi-gateway.cfg <<EOF
[config]
cluster_name = ceph
gateway_keyring = ceph.client.admin.keyring
api_secure = false
EOF
```

  * Copy file to all gateways

```shell
for h in ceph-mon{1,2,3}; do echo $h; scp iscsi-gateway.cfg $h:/etc/ceph; done
```

  * Restart and enable needed services on gateways

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h 'systemctl daemon-reload'; done
for h in ceph-mon{1,2,3}; do echo $h; ssh $h 'systemctl start tcmu-runner rbd-target-gw rbd-target-api; systemctl enable tcmu-runner rbd-target-gw rbd-target-api'; done
```

  * Setup target with the 3 portals (gateways) from "gwcli"

```shell
/> cd /iscsi-target 
/iscsi-target> create iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw
ok
/iscsi-target> cd iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw/gateways 
/iscsi-target...-igw/gateways> create ceph-mon1 192.168.122.21 skipchecks=true
OS version/package checks have been bypassed
Adding gateway, sync'ing 0 disk(s) and 0 client(s)
ok
/iscsi-target...-igw/gateways> create ceph-mon2 192.168.122.22 skipchecks=true
OS version/package checks have been bypassed
Adding gateway, sync'ing 0 disk(s) and 0 client(s)
ok
/iscsi-target...-igw/gateways> create ceph-mon3 192.168.122.23 skipchecks=true
OS version/package checks have been bypassed
Adding gateway, sync'ing 0 disk(s) and 0 client(s)
ok
```

  * Create a test disk (still from "gwcli")

```shell
/iscsi-target...-igw/gateways> cd /disks
/disks> create pool=rbd image=disk_1 size=1G
ok
```

  * Register a test host (initiator) with no auth, and assign the previously created disk

```shell
/disks> cd /iscsi-target/iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw/
/iscsi-target...-gw:iscsi-igw> cd hosts 
/iscsi-target...csi-igw/hosts> create iqn.1994-05.com.redhat:ceph-test
ok
/iscsi-target...hat:ceph-test> auth nochap
ok
/iscsi-target...hat:ceph-test> disk add rbd.disk_1
ok
```

  * Now everything is ready in the gateways. You can check configuration with:

```shell
gwcli ls
```

## iSCSI client

Now we will configure the ceph-test host as a iscsi initiatior to test multipath and failover

  * Connect to the ceph-test vm (192.168.122.12)

```shell
ssh root@192.168.122.12
```

  * Setup dns nameserver and gateway (and reboot)

```shell
echo "nameserver 192.168.122.1" > /etc/resolv.conf
echo "GATEWAY=192.168.122.1" >> /etc/sysconfig/network-scripts/ifcfg-eth0; reboot
```

  * When the ceph-test vm is back online, connect again with SSH

```shell
ssh root@192.168.122.12
```

  * Install needed software

```shell
yum install -y device-mapper-multipath iscsi-initiator-utils 
```

  * Setup multipath for ALUA failover

```
cat > /etc/multipath.conf <<EOF
devices {
        device {
                vendor                 "LIO-ORG"
                hardware_handler       "1 alua"
                path_grouping_policy   "failover"
                path_selector          "queue-length 0"
                failback               60
                path_checker           tur
                prio                   alua
                prio_args              exclusive_pref_bit
                fast_io_fail_tmo       25
                no_path_retry          queue
        }
}
EOF
```

  * Restart multipathd

```shell
systemctl restart multipathd
```

  * Set initiatorname to the one defined previously (/etc/iscsi/initiatorname.iscsi)

```shell
echo "InitiatorName=iqn.1994-05.com.redhat:ceph-test" > /etc/iscsi/initiatorname.iscsi
```

  * And restart iscsi service

```shell
systemctl restart iscsi
```

  * Scan iSCSI targets

```shell
iscsiadm -m discovery -t sendtargets -p 192.168.122.21
```

  * Login

```shell
iscsiadm -m node -T iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw --login
```

  * Now we should see a multipath device with 3 paths. Only one of them should be active

```shell
[root@ceph-test ~]# multipath -l
3600140508f8ad6c1f3d496abc91d6617 dm-0 LIO-ORG ,TCMU device
size=1.0G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='queue-length 0' prio=0 status=active
| `- 3:0:0:0 sdb 8:16 active undef unknown
|-+- policy='queue-length 0' prio=0 status=enabled
| `- 4:0:0:0 sdc 8:32 active undef unknown
`-+- policy='queue-length 0' prio=0 status=enabled
  `- 2:0:0:0 sda 8:0  active undef unknown
```

  * Create a XFS filesystem and mount it

```shell
mkfs -t xfs /dev/mapper/3600140508f8ad6c1f3d496abc91d6617
mount /dev/mapper/3600140508f8ad6c1f3d496abc91d6617 /mnt
```

  * Create a test file

```shell
echo "TEST1" > /mnt/__Test_file.txt
```

### Failover test

  * Make sure you have the tree paths enabled (but only one active)

```shell
[root@ceph-test ~]# multipath -l
3600140508f8ad6c1f3d496abc91d6617 dm-0 LIO-ORG ,TCMU device
size=1.0G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='queue-length 0' prio=0 status=active
| `- 3:0:0:0 sdb 8:16 active undef unknown
|-+- policy='queue-length 0' prio=0 status=enabled
| `- 4:0:0:0 sdc 8:32 active undef unknown
`-+- policy='queue-length 0' prio=0 status=enabled
  `- 2:0:0:0 sda 8:0  active undef unknown
```

  * Check which gateway is the currently active one. Look for "Owner" in the output of "gwcli ls"

```
     o- hosts .......................................................................................................... [Hosts: 1]
        o- iqn.1994-05.com.redhat:ceph-test ................................................. [LOGGED-IN, Auth: None, Disks: 1(1G)]
          o- lun 0 ............................................................................. [rbd.disk_1(1G), Owner: ceph-mon2]
```

  * In this example, the active gateway is **ceph-mon2**
  * Kill **ceph-mon2** (no "clean" shutdown, but "hard" poweroff)
  * Try to create a new file

```shell
echo "TEST2" > /mnt/__Test_file2.txt
```

  * I/O should be "freezed" for some seconds
  * After the configured timeout, a new path should be activated and the previous command should complete

  * Check the status of the multipath. Now you should see a new active patch, and the previous one marked as "failed"



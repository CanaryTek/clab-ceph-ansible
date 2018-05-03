# CloudLab: Deploy a Ceph Cluster on CentOS 7 using ceph-ansible

In this CloudLab we deploy a Ceph cluster using CentOS 7 and ceph-ansible

You can also check [this project](https://github.com/CanaryTek/clab-ceph), it's a **more complete and powerfull** implementation based on openSUSE and DeepSea

## The Lab

This lab consists of 9 VM based on CentOS 7 with static IP

| Name | IP | Resources | Disc | Services | Description | 
|------|----|-----------|------|----------|-------------|
| ceph-deploy | 192.168.122.11 | 1GB, cores | 1x40G | ansible deploy, ceph-dash | Deploy and monitoring host |
| ceph-test | 192.168.122.12 | 3GB, 1 core | 1x40G | None | Just a test host to use as a client for iSCSI, NFS, etc |
| ceph-mon{1,2,3} | 192.168.122.2{1,2,3}) | 1GB, 1 core| 1x40G | MON, MGR, MDS, iSCSI Gateway, NFS-Ganesha, RadosGW | Monitors and service gateways |
| ceph-osd{1,2,3,4} | 192.168.122.3{1,2,3,4}) | 1GB, 1 core | 1x40G + 2x30G | OSD | Ceph OSD storage hosts |

NOTE: As you can see, you will need a pretty powerfull computer to run all VM needed by this lab

## Common workflow

  0. Prepare the lab (download needed images)

```
rake prepare_lab
```

  1. Edit the lab.yml file to adapt to your needs (if needed)

  2. If using btrfs, create a "vm" subvolume so you can create snapwhot at each Lab stage

```
btrfs subvolume create vm
```

  3. Initialize the virtual machines for the lab

```
rake init_vms
```

## Common tasks

  1. Start/Stop all VM belonging to the lab

```
rake start_vms
rake stop_vms
```

  2. Create snapshot at an important milestone (i.e. after installation)

```
btfs snap -r vm .snapshots/01_after_installation
```

  3. Revert to the previous snapshot

```
btrfs subvol delete vm
btfs snap .snapshots/01_after_installation vm
```

## Labs

  * [Lab 1: Deploy_Ceph_Cluster](labs/01_Deploy_Ceph_Cluster.md)
  * [Lab 2: Monitoring](labs/02_Monitoring.md)


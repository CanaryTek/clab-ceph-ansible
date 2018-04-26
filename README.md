# CloudLab: Deploy a Ceph Cluster on CentOS 7 using ceph-ansible

In this CloudLab we deploy a Ceph cluster using CentOS 7 and ceph-ansible


## The Lab

This lab consists of 9 VM based on CentOS 7 with static IP

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

  * [Lab 1: Deploy_Ceph](labs/01_Deploy_Ceph/)


# Desploy Ceph cluster

## Cluster preparation

  * Add the following entries to /etc/hosts (change the IP prefix if needed)

```shell
cat >> /etc/hosts <<EOF
192.168.122.11  ceph-deploy
192.168.122.12  ceph-test
192.168.122.21  ceph-mon1
192.168.122.22  ceph-mon2
192.168.122.23  ceph-mon3
192.168.122.31  ceph-osd1
192.168.122.32  ceph-osd2
192.168.122.33  ceph-osd3
192.168.122.34  ceph-osd4
EOF
```

  * Copy /etc/hosts to all nodes

```shell
for h in ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; scp /etc/hosts root@$h:/etc/hosts; done
```

  * Disable ipv6 in all nodes (to avoid long delays)

```shell
cat > /etc/sysctl.d/disable-ipv6.conf <<EOF
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
for h in ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; scp /etc/sysctl.d/disable-ipv6.conf root@$h:/etc/sysctl.d/disable-ipv6.conf; done
```

  * Setup nameserver in all hosts

```shell
echo "nameserver 192.168.122.1" > /etc/resolv.conf
for h in ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'echo "nameserver 192.168.122.1" > /etc/resolv.conf'; done
```

  * For some reason cloud-init does not setup default gateway in this image. Set it manually and reboot

```shell
for h in ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'echo "GATEWAY=192.168.122.1" >> /etc/sysconfig/network-scripts/ifcfg-eth0; reboot'; done
echo "GATEWAY=192.168.122.1" >> /etc/sysconfig/network-scripts/ifcfg-eth0; reboot
```

  * Install and start NTP in all hosts

```shell
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h 'yum install -y ntp'; done
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h "ntpdate pool.ntp.org; systemctl start ntpd; systemctl enable ntpd "; done
```

## Install ceph-ansible in deploy host

  * Install ansible and git and clone ceph-ansible repo

```shell
yum install -y ansible git
cd /opt
git clone https://github.com/ceph/ceph-ansible.git
cd ceph-ansible
git checkout stable-3.0
```

  * Setup ceph_hosts inventory file

```
[mgrs]
ceph-mon1
ceph-mon2
ceph-mon3

[mons]
ceph-mon1
ceph-mon2
ceph-mon3

[mdss]
ceph-mon1
ceph-mon2
ceph-mon3

[rgws]
ceph-mon1
ceph-mon2
ceph-mon3

[nfss]
ceph-mon1
ceph-mon2
ceph-mon3

[osds]
ceph-osd1 ceph_crush_rack=rack1
ceph-osd2 ceph_crush_rack=rack2
ceph-osd3 ceph_crush_rack=rack2
ceph-osd4 ceph_crush_rack=rack4
```

  * Copy site.yml.sample to site.yml

  * Setup variables group_vars/all.yml

```yaml
---
# Variables here are applicable to all host groups NOT roles
ceph_origin: repository
ceph_repository: community
ceph_stable_release: luminous
public_network: "192.168.122.0/24"
cluster_network: "192.168.123.0/24"
monitor_interface: eth0
devices:
  - '/dev/vdb'
  - '/dev/vdc'
osd_scenario: collocated
osd_objectstore: bluestore
radosgw_interface: eth0
```

  * Run playbook

```shell
ansible-playbook site.yml -i ceph_hosts
```

### Complete stop/start

If you need to stop the whole cluster, follow this procedure to do a clean shutdown

#### Stop

  1. Stop the clients from using the RBD images/Rados Gateway on this cluster or any other clients.
  2. The cluster must be in healthy state before proceeding.
  3. Set the noout, norecover, norebalance, nobackfill, nodown and pause flags

```
ceph osd set noout
ceph osd set norecover
ceph osd set norebalance
ceph osd set nobackfill
ceph osd set nodown
ceph osd set pause
```

  4. Shutdown osd nodes one by one
  5. Shutdown monitor nodes one by one
  6. Shutdown admin node

#### Start

  1. Power on the admin node
  2. Power on the monitor nodes
  3. Power on the osd nodes
  4. Wait for all the nodes to come up , Verify all the services are up and the connectivity is fine between the nodes.
  5. Unset all the noout,norecover,noreblance, nobackfill, nodown and pause flags.

```
ceph osd unset noout
ceph osd unset norecover
ceph osd unset norebalance
ceph osd unset nobackfill
ceph osd unset nodown
ceph osd unset pause
```

  6. Check and verify the cluster is in healthy state, Verify all the clients are able to access the cluster.


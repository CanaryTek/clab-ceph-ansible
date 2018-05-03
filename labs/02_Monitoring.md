# Monitoring

## Monitoring with ceph-dash

We will install a docker based ceph-dash in the deploy host

  * Install docker

```shell
yum -y install docker
```

  * Start docker service

```shell
systemctl start docker
```

  * Copy ceph config and admin keyring from one of the nodes

```shell
scp -r ceph-mon1:/etc/ceph /etc
```

  * Run ceph-dash (config keyring and MON hosts)

```shell
docker run -p 5000:5000 -e CEPHMONS='192.168.122.21,192.168.122.22,192.168.122.23' -e KEYRING="$(sudo cat /etc/ceph/ceph.client.admin.keyring)" crapworks/ceph-dash:latest
```

  * Now you can connect to ceph-dash at http://192.168.122.11:5000/


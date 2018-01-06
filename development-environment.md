
#Development environment

If you are wanting a BTrDB setup on a workstation or laptop just to develop against it, this section will show you how to set up an analogue of the production setup, minus the ingress daemons.

## Prerequisites

This assumes you have docker installed and that your shell is `bash`.

## Configuring devmachine

Clone the smartgridstore repository and change into the devmachine directory

```
git clone https://github.com/BTrDB/smartgridstore
cd smartgridstore/devmachine
```

If necessary, edit the `environment.sh` file to parameterize the development environment. The defaults should work okay on Ubuntu and OS X. 

## Setting up the cluster

Source the environment file from the devmachine directory and run `start_devmachine.sh`

```
source ./environment.sh
sudo -E ./start_devmachine.sh
```

Your output should look like this:

```
[INFO] docker network created
[INFO] pulling containers
[INFO][PULL] Using default tag: latest
[INFO][PULL] latest: Pulling from btrdb/cephdaemon
[INFO][PULL] Digest: sha256:bc03572d198e784e2aebc78473920f777d56faeba909372c466530e6dc13ce4a
[INFO][PULL] Status: Image is up to date for btrdb/cephdaemon:latest
[INFO][PULL] latest: Pulling from btrdb/stubetcd
[INFO][PULL] Digest: sha256:6e871f7da2bbc5e717f82c2ac8529a9d58995fc3f937a0d1e49fb191f5232d39
[INFO][PULL] Status: Image is up to date for btrdb/stubetcd:latest
[INFO][PULL] 4.7.0: Pulling from btrdb/db
[INFO][PULL] Digest: sha256:91ca6adcc32795c810cc2bcfd3f6961c5d672e05d738127932e7f208189dffbf
[INFO][PULL] Status: Image is up to date for btrdb/db:4.7.0
[INFO][PULL] 4.7.0: Pulling from btrdb/apifrontend
[INFO][PULL] Digest: sha256:fb51a22aab7335753be78799dae69954f49c1287faa93fdb51bb570b9618c4ed
[INFO][PULL] Status: Image is up to date for btrdb/apifrontend:4.7.0
[INFO] waiting for monitor container to start
[OKAY] ceph config found
[WARN] custom monitor configs and restarting mon
[INFO] ceph MGR started
[INFO] ceph OSD 0 started
[INFO] ceph OSD 1 started
[INFO] ceph OSD 2 started
[INFO] ceph OSD 3 started
[INFO] etcd started
[INFO] checking pools
[INFO][POOL CREATE] pool 'btrdbhot' created
[INFO][POOL CREATE] pool 'btrdbcold' created
[INFO] checking database is initialized
[INFO][DB INIT] + ls /etc/ceph
[INFO][DB INIT] ceph.client.admin.keyring  ceph.conf  ceph.mon.keyring
[INFO][DB INIT] + set +x
[INFO][DB INIT] nameserver 127.0.0.11
[INFO][DB INIT] options ndots:0
[INFO][DB INIT] ensuring database
[INFO][DB INIT] [INFO]main.go:41 > Starting BTrDB version 4.7.0 

DB INIT output trimmed a bit

[INFO][DB INIT] Creating blank cold allocator
[INFO][DB INIT] Initializing cold pool
[INFO][DB INIT] Initializing hot pool
[INFO][DB INIT] Done
[INFO] waiting for DB server to start (20s)
[INFO] database server started
[INFO] admin console started
[INFO] plotter server started
[COMPLETE] =========================
Plotter is on https://127.0.0.1:8888
Console is on ssh://127.0.0.1:2222
BTrDB GRPC api is on 127.0.0.1:4410
BTrDB HTTP api is on http://127.0.0.1:9000
BTrDB HTTP swagger UI is on http://127.0.0.1:9000/swag
```

You can now access BTrDB and Mr. Plotter as if it were running on localhost.

## Interacting with Ceph

When you source the `environment.sh` file, it also adds `dvceph` and `dvrados` commands to your path. These commands act like the `ceph` and `rados` commands but appropriately execute inside the docker containers so that they connect to the devmachine ceph cluster. For example

```
$ dvceph -s  
cluster:
    id:     cf4e08c9-f0b2-4957-a2b5-daf5ae12609e
    health: HEALTH_WARN
            4 nearfull osd(s)
            application not enabled on 2 pool(s)
            too few PGs per OSD (24 < min 30)
            mon bb1943fcac64 is low on available space
 
  services:
    mon: 1 daemons, quorum bb1943fcac64
    mgr: 1849d337a4ea(active)
    osd: 4 osds: 4 up, 4 in
         flags nearfull
 
  data:
    pools:   2 pools, 32 pgs
    objects: 2 objects, 16 bytes
    usage:   756 GB used, 117 GB / 873 GB avail
    pgs:     32 active+clean

$ dvrados df
POOL_NAME USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS RD   WR_OPS WR   
btrdbcold    8       1      0      3                  0       0        0      3 2048      4 2048 
btrdbhot     8       1      0      3                  0       0        0      3 2048      4 2048 

total_objects    2
total_used       756G
total_avail      117G
total_space      873G

```

## Tearing down the cluster

When you are done, you can tear down the development machine with

``
source ./environment.sh 
sudo -E ./teardown_devmachine.sh 
```

Your output should look like

```
[INFO] removing container devmachine-ceph-osd-0 if exists
 DELETED
[INFO] removing container devmachine-ceph-osd-1 if exists
 DELETED
[INFO] removing container devmachine-ceph-osd-2 if exists
 DELETED
[INFO] removing container devmachine-ceph-osd-3 if exists
 DELETED
[INFO] removing container devmachine-ceph-mon if exists
 DELETED
[INFO] removing container devmachine-ceph-mgr if exists
 DELETED
[INFO] removing container devmachine-etcd if exists
 DELETED
[INFO] removing container devmachine-btrdbd if exists
 DELETED
[INFO] removing container devmachine-console if exists
 DELETED
[INFO] removing container devmachine-apifrontend if exists
 DELETED
[INFO] removing container devmachine-mrplotter if exists
 DELETED
[INFO] removing network cephnet if exists
 DELETED
[INFO] removing all state
[OKAY] all done here, have a great day!
```

## Limitations

The development environment is not particularly durable, so should not be used to store real data. 
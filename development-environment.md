# Development environment

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
[INFO][PULL] 4.15.3: Pulling from btrdb/db
[INFO][PULL] Digest: sha256:7a67fb58ceace27c3eb1e678afe37f0c5fbc6cd5dd0a1c5f8c4e01efe04d0040
[INFO][PULL] Status: Image is up to date for btrdb/db:4.15.3
[INFO][PULL] 4.15.3: Pulling from btrdb/apifrontend
[INFO][PULL] Digest: sha256:9ffc8a08dae7e8243561fc7ef5c6cda641a3bc18d8e755e841ddb9d1362d3dc3
[INFO][PULL] Status: Image is up to date for btrdb/apifrontend:4.15.3
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
[INFO][POOL CREATE] pool 'btrdbjournal' created
[INFO] checking database is initialized
[INFO][DB INIT] + ls /etc/ceph
[INFO][DB INIT] ceph.client.admin.keyring  ceph.conf  ceph.mon.keyring
[INFO][DB INIT] + set +x
[INFO][DB INIT] search san.rr.com
[INFO][DB INIT] nameserver 127.0.0.11
[INFO][DB INIT] options ndots:0
[INFO][DB INIT] ensuring database
[INFO][DB INIT] [INFO]main.go:41 > Starting BTrDB version 4.15.3 
[INFO][DB INIT] [INFO]main.go:50 > TRACING IS _NOT_ ENABLED
[INFO][DB INIT] Connecting to ETCD with 3 endpoints. 
[INFO][DB INIT] EPZ:([]string{"http://172.29.0.20:2379", "http://172.29.0.20:2379", "http://172.29.0.20:2379"})
[INFO][DB INIT] [WARNING]etcd.go:80 > No global etcd config found, bootstrapping
[INFO][DB INIT] [WARNING]etcd.go:97 > No etcd config for this node (3a7e11100ecf-000) found, bootstrapping
[INFO][DB INIT] [INFO]main.go:82 > CONFIG OKAY!
[INFO][DB INIT] [INFO]main.go:90 > Ensuring database is initialized
[INFO][DB INIT] reading ceph config: /etc/ceph/ceph.conf hotpool=btrdbhot coldpool=btrdbcold
[INFO][DB INIT] connection OK, opening cold IO context
[INFO][DB INIT] cold OK, opening hot IO context
[INFO][DB INIT] checking for cold allocator
[INFO][DB INIT] checking for legacy allocator
[INFO][DB INIT] Creating blank cold allocator
[INFO][DB INIT] Initializing cold pool
[INFO][DB INIT] onto stage2
[INFO][DB INIT] Initializing hot pool
[INFO][DB INIT] [INFO]main.go:92 > Done
[INFO] waiting for DB server to start (20s)
[INFO] waiting for DB server to start (10s)
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
    id:     ecf0a5d9-d29a-4f16-bdd2-dbe92e0de8ed
    health: HEALTH_WARN
            application not enabled on 3 pool(s)
 
  services:
    mon: 1 daemons, quorum 289c432a78ee
    mgr: 2f35c4ae3fc6(active)
    osd: 4 osds: 4 up, 4 in
 
  data:
    pools:   3 pools, 48 pgs
    objects: 3 objects, 24 bytes
    usage:   562 GB used, 2003 GB / 2565 GB avail
    pgs:     48 active+clean

$ dvrados df
POOL_NAME    USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS RD   WR_OPS WR   
btrdbcold       8       1      0      3                  0       0        0      3 2048      4 2048 
btrdbhot        8       1      0      3                  0       0        0      3 2048      4 2048 
btrdbjournal    8       1      0      3                  0       0        0      0    0      1 1024 

total_objects    3
total_used       562G
total_avail      2003G
total_space      2565G

```

## Tearing down the cluster

When you are done, you can tear down the development machine with

```
source ./environment.sh 
sudo -E ./teardown_devmachine.sh
```

Your output should look like

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

The development environment is not particularly durable, so should not be used to store real data. You can make the data more durable by setting

```
USE_EPHEMERAL_STORAGE=N
```




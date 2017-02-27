
#Development environment

If you are wanting a BTrDB setup on a workstation or laptop just to develop against it, this guide will show you how to set up an analogue of the production setup, including kubernetes, on a single machine.

## Prerequisites

This assumes you have docker and virtualbox installed on your target machine.

## Setting up Ceph

Lets create a MON and four OSDs using docker. We will place them on a new docker network so that we can hard code their IPs and know they won't collide. The data will be stored in directories in /srv/ceph


```
mkdir -p /srv/ceph

docker network create --subnet 172.25.0.0/24 cephnet

docker run -d --net cephnet --ip 172.25.0.5 \
 --pid=host \
 -v /srv/ceph/etc/ceph:/etc/ceph \
 -v /srv/ceph/var/lib/ceph/:/var/lib/ceph/ \
 -e MON_IP=172.25.0.5 \
 -e CEPH_PUBLIC_NETWORK=172.25.0.0/24 \
 ceph/daemon mon

docker run -d --net cephnet --ip 172.25.0.6 \
 --pid=host \
 -v /srv/ceph/etc/ceph:/etc/ceph \
 -v /srv/ceph/var/lib/ceph/:/var/lib/ceph/ \
 -v /srv/ceph/osd:/var/lib/ceph/osd \
 -e OSD_TYPE=directory \
 ceph/daemon osd

docker run -d --net cephnet --ip 172.25.0.7 \
 --pid=host \
 -v /srv/ceph/etc/ceph:/etc/ceph \
 -v /srv/ceph/var/lib/ceph/:/var/lib/ceph/ \
 -v /srv/ceph/osd:/var/lib/ceph/osd \
 -e OSD_TYPE=directory \
 ceph/daemon osd

docker run -d --net cephnet --ip 172.25.0.8 \
 --pid=host \
 -v /srv/ceph/etc/ceph:/etc/ceph \
 -v /srv/ceph/var/lib/ceph/:/var/lib/ceph/ \
 -v /srv/ceph/osd:/var/lib/ceph/osd \
 -e OSD_TYPE=directory \
 ceph/daemon osd

docker run -d --net cephnet --ip 172.25.0.9 \
 --pid=host \
 -v /srv/ceph/etc/ceph:/etc/ceph \
 -v /srv/ceph/var/lib/ceph/:/var/lib/ceph/ \
 -v /srv/ceph/osd:/var/lib/ceph/osd \
 -e OSD_TYPE=directory \
 ceph/daemon osd
```


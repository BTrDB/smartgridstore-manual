# Installation

The installation process has the following stages:

* [Create ceph pools for SGS](#creating-ceph-pools)
* [Create siteconfig.yaml](#site-config)
* [Generate your kubernetes config files](#generating-manifest-files)
* [Starting everything up](#startup)

## Creating ceph pools

SGS requires four ceph pools. Although these can be the same pool with no problems,   
having them as different pools will allow you to migrate   
onto different storage later without any difficulty.

The first two pools are the ones that BTrDB uses to store data. The pools can have any  
names, you will specify the pool names in the site config in the next step. If you have a fair bit of SSD storage in your cluster it is recommended to place the `btrdb_hot` pool on SSD.

```bash
ceph osd pool create btrdb_hot 256 256 
ceph osd pool create btrdb_data 256 256
```

The next pool is for the BTrDB journal. This SHOULD be on SSD, but it won't ever grow particularly big. A pool of about 100GB should suffice even in production. You can put this on spinning metal, but not in production.

```
ceph osd pool create btrdb_journal 256 256
```

The final pool is used by the `receiver/ingester` daemon pair for storing incoming PMU data in its raw form before  
it is processed by BTrDB. This stage is skipped if you use the `pmu2btrdb` ingress daemon.

```
ceph osd pool create staging 256 256
```

For details on the `ceph osd pool` command, such as how to specify the placement ruleset, consult the [ceph documentation](http://docs.ceph.com/docs/luminous/rados/operations/pools/).  
In particular, a placement group count of 256 is appropriate for between 5 and 10 OSDs assuming you are creating four  
pools.

## Site config

Although all site-specific information can be edited in the kubernetes manifest files, it can be tedious  
to copy and paste common information between multiple files. To make this a little easier, there is a tool,  
`mfgen`, that will take a site configuration file and generate all the manifest files for you.

To begin, download the `mfgen` for the version of smartgridstore that you are installing. You can get this  
from the [releases page on github](https://github.com/BTrDB/smartgridstore/releases).

To generate an example siteconfig, run

```bash
./mfgen mksiteconf
```

You should get a file similar to this. The file that mfgen creates is correct for the version of smartgridstore. If  
the file differs from what you see on this guide, it is likely that this guide is out of date, do not change the generated  
file.

```yaml
apiVersion: smartgrid.store/v1
kind: SiteConfig
# this is the kubernetes namespace that you are deploying into
targetNamespace: sgs
containers:
  #this can be 'development' or 'release'
  channel: release
  imagePullPolicy: Always
siteInfo:
  ceph:
    # the staging pool is where ingress daemons stage data. It can be
    # ignored if you are not using ingress daemons
    stagingPool: staging

    # the btrdb data pool (or cold pool) is where most of the data
    # is stored. It is typically large and backed by spinning metal drives
    btrdbDataPool: btrdb_data
    # the btrdb hot pool is used for more performance-sensitive
    # data. It can be smaller and is usually backed by SSDs
    btrdbHotPool: btrdb_hot
    # the btrdb journal pool is small and used to aggregate write
    # load into better-performing batches. It should be backed by
    # a very high performance pool
    btrdbJournalPool: btrdb_journal
    # If you are using Rook, the recommended way to connect to ceph
    # is to mount the config file directly. Uncomment this to do so
    # configFile: /var/lib/rook/rook-ceph/rook-ceph.config
    # mountConfig: true

  etcd:
    # which nodes should run etcd servers. There must be three entries
    # here, so if you have fewer nodes, you can have duplicates. The node
    # names must match the output from 'kubectl get nodes
    nodes:
    - host0
    - host1
    - host2
    # which version of etcd to deploy
    version: v3.3.5

  # the external IPs listed here are where the services can be contacted
  # e.g for the plotter or the BTrDB API
  externalIPs:
  - 123.123.123.1
```

You can change channel to be "development" if you want to use the development images in the cluster. `release` images never change, and contain software that we believe is stable. `dev` images contain our work-in-progress for the next release version so may not work or may change frequently on the same version tag. To allow this to work, your imagePullPolicy must be "Always" for dev images, but it can be "IfNotPresent" on `release` images if desired.

Under siteInfo, fill in the names of the ceph pools that you created so far. In addition, fill in the external IP addresses of all the servers in the cluster. These will be put into the externalIPs field of services that are meant to be public facing, such as the ingress daemons or the plotter.

If you are using Rook, then make sure to uncomment `configFile` and `mountConfig` .

## Generating manifest files

Once you have configured the siteconfig file, run

```bash
./mfgen generate siteconfig.yaml
```

This will create a directory called `manifests` which contains the kubernetes config files for all the services in the SGS stack.

At the time the guide was written, this directory looks like

```bash
./core/etcd-cluster.daemonset.yaml
./core/create_admin_key.sh
./core/secret_ceph_keyring.sh
./core/ensuredb.job.yaml
./core/btrdb.statefulset.yaml
./core/adminconsole.deployment.yaml
./core/apifrontend.deployment.yaml
./core/mrplotter.deployment.yaml
./ingress/gepingress.deployment.yaml
./ingress/pmu2btrdb.deployment.yaml
./ingress/receiver.deployment.yaml
./ingress/c37ingress.deployment.yaml
./ingress/ingester.deployment.yaml
```

These core config files need to be applied in a specific order:

## Core

First install the the core services

Begin with the secrets \(if you are using rook you can skip this\).

```
cd core
chmod a+x secret_ceph_keyring.sh
./secret_ceph_keyring.sh
```

Then create the etcd cluster

```
cd core
kubectl create -f etcd-cluster.daemonset.yaml
```

Then wait for the etcd pods to exist by checking for the three etcd pods with `kubectl get pods`

Now you can create the BTrDB database:

```
kubectl create -f ensuredb.job.yaml
```

You can then create the host keys for the admin console:

```
./create_admin_keys.sh
```

Then create the BTrDB statefulset and the rest of the core services

```
kubectl create -f btrdb.statefulset.yaml
kubectl create -f mrplotter.deployment.yaml
kubectl create -f apifrontend.deployment.yaml
kubectl create -f adminconsole.deployment.yaml
```

## Ingress daemons

After the core is up and running, you can create the necessary ingress daemons by running `kubectl create -f` on their manifests in the `ingress` directory

At this stage, your cluster is up and running, but you will likely want to drop into  
the admin console and change the default password and add PMU devices. Please consult the  
admin console section for more information.


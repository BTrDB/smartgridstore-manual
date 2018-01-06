# Installation

The installation process has the following stages:

- [Create ceph pools for SGS](#creating-ceph-pools)
- [Create siteconfig.yaml](#site-config)
- [Generate your kubernetes config files](#generating-manifest-files)
- [Starting everything up](#startup)

## Creating ceph pools

SGS requires three ceph pools. Although these can be the same
pool with no problems, having them as different pools will allow you
to migrate onto different storage later without any difficulty.

The first two pools are the ones that BTrDB uses to store data. The pools can have any
namse, you will specify the pool names in the site config in the next step.
The third pool is used by the `receiver/ingester` daemon pair for storing incoming PMU data in its raw form before
it is processed by BTrDB. This stage is skipped if you use the `pmu2btrdb` ingress daemon.

```bash
ceph osd pool create btrdb_hot 256
ceph osd pool create btrdb_data 256
ceph osd pool create staging 256
```

For details on this command, consult the [ceph documentation](http://docs.ceph.com/docs/luminous/rados/operations/pools/).
In particular, a placement group count of 256 is appropriate for between 5 and 10 OSDs assuming you are creating four
pools (one for the RBD pv's created in the prerequisites section and three in this section).

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
  #this can be 'development'
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
    # the RBD pool is used to provision persistent storage for
    # kubernetes pods. It can use spinning metal.
    rbdPool: fastrbd

  # the external IPs listed here are where the services can be contacted
  # e.g for the plotter or the BTrDB API
  externalIPs:
  - 123.123.123.1
```

You can change channel to be "development" if you want to use the development images in the cluster. `release` images never change, and contain software that we believe is stable. `dev` images contain our work-in-progress for the next release version so may not work or may change frequently on the same version tag. To allow this to work, your imagePullPolicy must be "Always" for dev images, but it can be "IfNotPresent" on `release` images if desired.

Under siteInfo, fill in the names of the ceph pools that you created so far. In addition, fill in the external IP addresses of all the servers in the cluster. These will be put into the externalIPs field of services that are meant to be public facing, such as the ingress daemons or the plotter.

## Generating manifest files

Once you have configured the siteconfig file, run

```bash
./mfgen generate siteconfig.yaml
```

This will create a directory called `manifests` which contains the kubernetes config files for all the services in the SGS stack.

At the time the guide was written, this directory looks like

```bash
$ cd manifests && find .
./readme.md
./global
./global/etcd.clusterrole.yaml
./core
./core/etcd-operator.deployment.yaml
./core/create_admin_key.sh
./core/secret_ceph_keyring.sh
./core/etcd.cluster.yaml
./core/etcd.clusterrolebinding.yaml
./core/ensuredb.job.yaml
./core/adminconsole.deployment.yaml
./core/mrplotter.deployment.yaml
./core/etcd.serviceaccount.yaml
./core/apifrontend.deployment.yaml
./core/btrdb.statefulset.yaml
./ingress
./ingress/ingester.deployment.yaml
./ingress/pmu2btrdb.deployment.yaml
./ingress/receiver.deployment.yaml
./ingress/c37ingress.deployment.yaml
```

These config files need to be applied in order.

## Global

If you are upgrading an existing BTrDB cluster, or installing a new BTrDB cluster on the same kubernetes cluster (different namespace), you can skip this part. If this is the first time on this kubernetes/ceph cluster, you need to do a little bootstrapping.

```
cd global
kubectl create -f global/etcd.clusterrole.yaml
```

## Core

Now it is time to install the core services

Begin with the secrets (if you did not already create these manually in the prerequisites section)

```
cd core
chmod a+x secret_ceph_keyring.sh
./secret_ceph_keyring.sh
```

Then create the etcd cluster

```
cd core
kubectl create -f etcd.serviceaccount.yaml
kubectl create -f etcd.clusterrolebinding.yaml
kubectl create -f etcd-operator.deployment.yaml
kubectl create -f etcd.cluster.yaml
```

Then wait for the etcd pods to exist by checking for the three etcd pods with `kubectl get pods`

Now you can create the BTrDB database:

```
kubectl create -f ensuredb.job.yaml 
```

Create the BTrDB statefulset and the rest of the core services

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

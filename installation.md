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

Begin with the secrets

```
cd core
chmod a+x secret_ceph_keyring.sh
./secret_ceph_keyring.sh
```

Begin with the etcd cluster

```
cd core
kubectl create -f etcd.clusterrolebinding.yaml
kubectl create -f etcd-operator.deployment.yaml
kubectl create -f etcd.cluster.yaml
```

Then wait for the etcd pods to exist by checking for the three etcd pods with `kubectl get pods`

Then create the keys

```


## Startup



The admin console uses an SSH host key to prevent man-in-the-middle attacks. You have to create this host key as a kubernetes secret.
It cannot be auto-generated because then it would not persist across upgrades or pod rescheduling. To create the key, run

```bash
source manifests/create_admin_key.sh
```

Next, the BTrDB database needs to be created. This requires etcd to be running, so start those first:

```bash
kubectl create -f manifests/etcd.statefulset.yaml
```

Watch with `kubectl get pods` until all three etcd pods are up and running. It may take some time as it pulls the containers to your servers. Once you see output like this:

```
NAME                            READY     STATUS        RESTARTS   AGE
etcd-0                          1/1       Running       0          24s
etcd-1                          1/1       Running       0          17s
etcd-2                          1/1       Running       0          14s
```

You can create the database:

```
kubectl create -f manifests/createdb.job.yaml
```

The pod will terminate upon success, so to check it ran ok, run `kubectl get jobs`. You should see

```
kubectl get jobs
NAME                  DESIRED   SUCCESSFUL   AGE
btrdb-createdb        1         1            44s
```

Which shows that the job ran successfully.

At this point you can start BTrDB and scale it up
```
kubectl create -f manifests/btrdb.statefulset.yaml
kubectl scale --replicas=3 statefulset btrdb
```

You can also start the rest of the services
```
kubectl create -f manifests/adminconsole.deployment.yaml
kubectl create -f manifests/ingester.deployment.yaml
kubectl create -f manifests/receiver.deployment.yaml
kubectl create -f manifests/mrplotter.deployment.yaml
kubectl create -f manifests/pmu2btrdb.deployment.yaml
```

After a short while pulling containers, `kubectl get pods` should show you:

```
NAME                             READY     STATUS        RESTARTS   AGE
btrdb-0                          1/1       Running       0          4m
btrdb-1                          1/1       Running       0          3m
btrdb-2                          1/1       Running       0          3m
console-2122947954-ztt80         1/1       Running       0          1m
etcd-0                           1/1       Running       0          8m
etcd-1                           1/1       Running       0          8m
etcd-2                           1/1       Running       0          8m
ingester-upmu-3112618582-ddnj2   1/1       Running       0          1m
mrplotter-1281955092-1dxz0       1/1       Running       0          1m
mrplotter-1281955092-3499n       1/1       Running       0          1m
pmu2btrdb-2484217541-dkzdw       1/1       Running       0          55s
pmu2btrdb-2484217541-s35th       1/1       Running       0          55s
pmu2btrdb-2484217541-t0l2t       1/1       Running       0          55s
pmu2btrdb-2484217541-zt3zk       1/1       Running       0          55s
receiver-upmu-3502962172-r6p3j   1/1       Running       0          1m
receiver-upmu-3502962172-wm4h6   1/1       Running       0          1m
```

You can do any changes to scaling that you want at this time. For example, the
default number of pmu2btrdb instances in v4.4.3 is 4, which seems too high. You
can correct this with

```
kubectl scale --replicas=2 deployment pmu2btrdb
```

If any of these pods are not running correctly, (for example STATUS is CrashLoopBackoff) you
can get more details by using `kubectl logs -f <podname>`. For example say you saw

```
NAME                             READY     STATUS              RESTARTS   AGE
ingester-upmu-3112618582-ddnj2   0/1       CrashLoopBackOff    1          17s
```

You can run `kubectl logs -f ingester-upmu-3112618582-ddnj2` which shows:

```
Booting ingester version 4.4.3
2017/02/20 22:28:26 Connecting to etcd...
Could not open ceph handle: rados: No such file or directory
```

This shows that it could not open the ceph pool because it does not exist. In this
case it was because the `staging` pool had not been created. After fixing the problem,
the pod will generally automatically start, but you can preempt any backoff time
by executing

```
kubectl delete pod -l app=ingester-upmu
```

At this stage, your cluster is up and running, but you will likely want to drop into
the admin console and change the default password and add PMU devices. Please consult the
admin console section for more information.

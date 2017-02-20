# Installation

The installation process has the following stages:

- [Create ceph pools for SGS](#creating-ceph-pools)
- [Create siteconfig.yaml](#site-config)
- [Generate your kubernetes config files](#generating-manifest-files)
- [Starting everything up](#startup)
- [Final configuration in the admin console](#final-config)

## Creating ceph pools

SGS requires two ceph pools. Although these can be the same
pool with no problems, having them as different pools will allow you
to migrate onto different storage later without any difficulty.

The first pool is the one that BTrDB uses to store data. For this release
both the hot and cold data is stored in the same pool. The pool can have any
name, you will specify the pool name in the site config in the next step.
The second pool is used by the `receiver/ingester` daemon pair for storing incoming PMU data in its raw form before
it is processed by BTrDB. This stage is skipped if you use the `pmu2btrdb` ingress daemon. More details
can be found in the [ingress daemons section](ingressdaemons.md).

```bash
ceph osd pool create btrdb 256
ceph osd pool create staging 256
```

For details on this command, consult the [ceph documentation](http://docs.ceph.com/docs/jewel/rados/operations/pools/).
In particular, a placement group count of 256 is appropriate for between 5 and 10 OSDs assuming you are creating three
pools (one for the RBD pv's created in the prequisites section and two in this section).

## Site config

Although all site-specific information can be edited in the kubernetes manifest files, it can be tedious
to copy and paste common information between multiple files. To make this a little easier, there is a tool,
`mfgen`, that will take a site configuration file and generate all the manifest files for you.

To begin, download the `mfgen` for the version of smartgridstore that you are installing. You can get this
from the [releases page on github](https://github.com/immesys/smartgridstore/releases).

To generate an example siteconfig, run

```bash
./mfgen mksiteconf
```

You should get a file similar to this. The file that mfgen creates is correct for the version of smartgridstore. If
the file differs from what you see on this guide, it is likely that the guide is out of date, do not change the generated
file.

```yaml
apiVersion: smartgrid.store/v1
kind: SiteConfig
containers:
  channel: release
  imagePullPolicy: Always
siteInfo:
  ceph:
    stagingPool: staging
    btrdbPool: btrdb
    rbdPool: rbd
  externalIPs:
  - 128.32.37.141
```

You can change channel to be "dev" if you want to use the development images in the cluster. `release` images never change, and contain software that we believe is stable. `dev` images contain our work-in-progress for the next release version so may not work or may change frequently on the same version tag. To allow this to work, your imagePullPolicy must be "Always" for dev images, but it can be "IfNotPresent" on `release` images if desired.

Under siteInfo, fill in the names of the ceph pools that you created so far. In addition, fill in the external IP addresses of all the servers in the cluster. These will be put into the externalIPs field of services that are meant to be public facing, such as the ingress daemons or the plotter.

## Generating manifest files

Once you have configured the siteconfig file, run

```bash
./mfgen generate siteconfig.yaml
```

This will create a directory called `manifests` which contains the kubernetes config files for all the services in the SGS stack.

At the time the guide was written, this directory looks like

```bash
$ ls manifests/
adminconsole.deployment.yaml
btrdb.statefulset.yaml
create_admin_key.sh
createdb.job.yaml
etcd.statefulset.yaml
ingester.deployment.yaml
mrplotter.deployment.yaml
pmu2btrdb.deployment.yaml
receiver.deployment.yaml
```

Don't apply these config files just yet, there is some bootstrapping to be done first.

## Startup

If you are upgrading an existing BTrDB cluster, you can skip this part. If this is the first time on this kubernetes/ceph cluster, you need to do a little bootstrapping.

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

## Final config

At this stage, your cluster is up and running, but you will likely want to drop into
the admin console and change the default password and add PMU devices. Please consult the
admin console section for more information.

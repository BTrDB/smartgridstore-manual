# Prerequisites

This guide assumes that you have ceph and kubernetes set up. Although there are many guides for getting to that point, we have found the following useful:

* [Ceph quick start](http://docs.ceph.com/docs/master/start/)
* [K8S with kubeadm](https://kubernetes.io/docs/getting-started-guides/kubeadm/)

BTrDB supports Rook as well, which is an easier way of managing ceph using a kubernetes operator. You can see more details here:

* [Rook Quickstart](https://rook.github.io/docs/rook/v0.8/quickstart-toc.html)

This chapter will walk you through a couple checks to ensure that those guides  
were successful and that your cluster is ready to proceed. It will also walk you  
through some additional configuration, namely:

* Creating and configuring the SGS k8s namespace
* Configuring the kubernetes secrets for ceph \(if you are doing bare metal ceph rather than rook\)

## NTP

Several parts of the stack, including ceph, work better if your servers have accurately synchronized time. You can check this by  
running

```bash
ntpq -p
```

You should get output like

```
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
+128.32.132.23 ( 169.229.60.110   3 u  896 1024   37   15.893    0.716   1.566
-cronus-132.CS.B 169.229.60.43    3 u  924 1024   37   14.365    0.017   1.201
 0.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 1.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 3.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
*x.ns.gin.ntt.ne 249.224.99.213   2 u  889 1024   37   12.326   -0.718   8.438
-www.calderamx.n 66.246.75.245    3 u  909 1024   37   13.631   -0.829   0.743
+ntp.newfxlabs.c 216.218.192.202  2 u 1025 1024    7   22.406    0.552   0.756
```

The way to read this, is first ignore any entries that have a `refid` of `POOL` or a `when` that is nonexistent or large. Then look at the offset column. This is in RMS milliseconds. If this is greater than about 10ms then you should sort out your time synchronization first before proceeding.

If `ntpq` is not found, you likely skipped over the Ceph preflight sections. Please make sure that you have ntp installed on your servers  
and that there is no clock slew before proceeding.

## Ceph

If you log into a node or master with ssh and run

```bash
sudo ceph -s
```

You should see some output similar to

```
cluster 36e53020-8ab4-41f5-abe7-19dea66db829
 health HEALTH_WARN
        too few PGs per OSD (11 < min 30)
 monmap e1: 3 mons at {compound-0=10.20.0.10:6789/0,compound-3=10.20.0.13:6789/0,compound-7=10.20.0.17:6789/0}
        election epoch 86, quorum 0,1,2 compound-0,compound-3,compound-7
 osdmap e1888: 72 osds: 72 up, 72 in
        flags sortbitwise,require_jewel_osds
  pgmap v991267: 272 pgs, 3 pools, 8794 GB data, 2502 kobjects
        26413 GB used, 210 TB / 235 TB avail
             271 active+clean
               1 active+clean+scrubbing+deep
client io 122 MB/s wr, 0 op/s rd, 171 op/s wr
```

In this case, we have not created all our pools yet so it is warning us that we have too few placement groups \(PGs\) per OSD, but this is not important. If you get an output like:

```
The program 'ceph' is currently not installed. You can install it by typing:
sudo apt install ceph-common
```

Do _NOT_ install ceph via apt as it could be an old version. Go back to the ceph tutorial and make sure that you have followed all the steps correctly. If you have installed ceph using Rook, you can install `ceph-common` using apt-get and then create a symlink from `/etc/ceph/ceph.conf` pointing to `/var/lib/rook/rook-ceph/rook-ceph.config`.

If you get an output like

```
2017-02-20 11:06:38.179700 7ff8a271e700 -1 auth: unable to find a keyring on /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin: (2) No such file or directory
2017-02-20 11:06:38.179718 7ff8a271e700 -1 monclient(hunting): ERROR: missing keyring, cannot use cephx for authentication
2017-02-20 11:06:38.179720 7ff8a271e700  0 librados: client.admin initialization error (2) No such file or directory
Error connecting to cluster: ObjectNotFound
```

This is actually not as bad as it looks. What it usually means is that you forgot the `sudo` part of the command, and your current user does not have permission to access the files in `/etc/ceph/` that authenticate the ceph client.

## Kubernetes

A working kubernetes stack not only has a master with multiple nodes available to schedule pods, but also has working networking provided by a cluster addon. We test using `weave` but in theory any networking addon will work.

The first thing to check is that your default namespace is correct. Execute

```bash
kubectl config get-contexts
```

You should get output similar to this:

```
CURRENT   NAME                 CLUSTER      AUTHINFO   NAMESPACE
*         admin@kubernetes     kubernetes   admin      sgs
          kubelet@kubernetes   kubernetes   kubelet
```

If you don't, you may not have a config for kubernetes yet, you can copy `/etc/kubernetes/admin.conf` to `~/.kube/config` and then you should get the above lines.

The important field is that the line with a `*` on it should have the namespace `sgs`. If this is not the case, execute:

```
kubectl create namespace sgs
CONTEXT=$(kubectl config view | awk '/current-context/ {print $2}')
kubectl config set-context $CONTEXT --namespace=sgs
```

To test if you have your environment properly set up, put the following in a file called `test.yaml` taking care to replace the list of externalIPs in the service with the list of your server's external IPs.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hostnames
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: gcr.io/google_containers/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: hostnames
  labels:
    app: hostnames
spec:
  ports:
  - port: 5643
    targetPort: 9376
  externalIPs:
  - 123.123.123.1
  - 123.123.123.2
  - 123.123.123.3
  selector:
    app: hostnames
```

Then run `kubectl apply -f test.yaml`. You should see

```
deployment "hostnames" created
service "hostnames" created
```

If you execute `kubectl get pods -o wide -n sgs` you should see something like

```
NAME                            READY     STATUS        RESTARTS   AGE       IP                NODE
hostnames-3799501552-1z4nb      1/1       Running       0          17s       192.168.82.177    compound-4
hostnames-3799501552-64v3g      1/1       Running       0          17s       192.168.193.116   compound-7
hostnames-3799501552-ww2gp      1/1       Running       0          17s       192.168.220.3     compound-2
```

These three pods have a cluster-ip \(shown in the IP column\) which is part of a CIDR only accessible within the cluster.  
We also created a load-balancing service, which has an IP in the cluster-ip CIDR:

```
kubectl get services
NAME           CLUSTER-IP       EXTERNAL-IP                   PORT(S)          AGE
hostnames      10.99.78.190     128.32.37.198,128.32.37.238   5643/TCP         15m
```

And it will also have the external IPs that you designated in the service file.  
To test that the full stack is working properly and the external IP is routing to the cluster-ip which is  
selecting pods and routing to the pod IPs, you can curl the service from a machine outside the cluster.  
If you curl it a couple times you should see that a different pod serves each request \(the hostnames image just serves  
the name of the pod over HTTP\):

    for i in `seq 1 10`; do curl 128.32.37.198:5643; done
    hostnames-3799501552-64v3g
    hostnames-3799501552-64v3g
    hostnames-3799501552-ww2gp
    hostnames-3799501552-64v3g
    hostnames-3799501552-1z4nb
    hostnames-3799501552-1z4nb
    hostnames-3799501552-ww2gp
    hostnames-3799501552-ww2gp
    hostnames-3799501552-ww2gp
    hostnames-3799501552-ww2gp

You can test each of the external IPs that you specified. All of them should work equally.

If you encountered some problems following along here, something is wrong with your kubernetes setup.  
Please consult the kubernetes documentation and resolve those issues before proceeding.

To clean up run

```
kubectl delete -f test.yaml
```

## Finishing up

At this point, your cluster should be ready to install BTrDB and SmartGrid Store. If you encountered any problems that you were unable  
to resolve, please contact btrdb@googlegroups.com.


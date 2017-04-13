# Troubleshooting

This contains a bit of troubleshooting info, as well as some tips and tricks. We will update this as we hear back from users about things they have run in to.

## I lost an etcd pod and now it won't start up

You may see a message like

```
raft: tocommit(234) is out of range [lastIndex(0)]. Was the raft log corrupted, truncated, or lost?
```

First off, fear not, there are multiple ways to recover from this. If it is only a minority of etcd pods that were lost (1/3 or 2/5) then the fix is very simple. You need to remove and re-add the member to let etcd know that it is ok to recover that member. As an example:

```
# etcd-1 was lost. etcd-0 and etcd-2 are healthy
$ kubectl exec -it etcd-0 /bin/bash
$ export ETCDCTL_API=3
$ etcdctl member list
63cdb3774b98cc2e, started, etcd-2, http://etcd-2.etcd:2380, http://etcd-2.etcd:2379
69691ed70da97612, started, etcd-0, http://etcd-0.etcd:2380, http://etcd-0.etcd:2379
7e44220b60d58d6a, started, etcd-1, http://etcd-1.etcd:2380, http://etcd-1.etcd:2379
# we want to remove the last one
$ etcdctl member remove 7e44220b60d58d6a 
Member 7e44220b60d58d6a removed from cluster 9c3a8eb86bab5e36
# now re-add it
$ etcdctl member add etcd-1 --peer-urls http://etcd-1.etcd:2380
```

The pod should now automatically start ok on the next try (it was likely in crashloopbackoff).

If you have lost a majority of etcd nodes, please immediately grab your etcd backups and store them in a safe place. The backups are taken every six hours and only the last few are kept. If you wait too long, the relevant backup could be rotated out and lost forever:

```
$ kubectl get pods | grep "etcd-backup"
etcd-backup-3266148700-fg6c4   1/1       Running   0          14h
$ kubectl cp etcd-backup-3266148700-fg6c4:/srv/persist etcd_backups
# your backups are now in the etcd_backups directory
```
If the pod are down, please map and mount the rbd used by the etcd-backup pod (if kubernetes is still working you can get the ID with
`kubectl get pv`. Once you mount the RBD, copy the files off as above.

Then contact btrdb@googlegroups.com for help. Since 4.4.8 the procedure for recovering etcd from backup has improved substantially, but we have not yet documented it.

The reason this happens is that in the current versions of smartgridstore we use `emptyDir` storage volumes for etcd pods. This is probably temporary and we may change this, but it is because our initial testing showed that RBDs did not perform well enough under the synchronous load of etcd. For machine reboots, this is not a problem, but for machine failures or pod deletions, this manual intervention is required. We are working on a way to safely automate this.

## I cannot access BTrDB from outside kubernetes pods

If you are trying to access BTrDB from a machine that is part of the kubernetes cluster, the best way is to configure your host DNS to check kubernetes' DNS first. We have done this by adding a line with the cluster DNS IP to `/etc/resolvconf/resolv.conf.d/head` like:

```
nameserver 10.96.0.10
```

But the details of this will depend on your kubernetes network addon (that determines the DNS IP) and your Linux distribution.

If you are trying to access BTrDB from outside the cluster entirely, unfortunately this is not a supported configuration in this version (although we are actively working on it). As a temporary stopgap, you can scale BTrDB down to a single node, add an externalIPs field to the btrdb-bootstrap service and configure the `BTRDB_APPEND_ADVERTISE_GRPC` environment variable in the BTrDB statefulset to `<externalip>:4410`.


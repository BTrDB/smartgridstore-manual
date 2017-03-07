# Troubleshooting

This contains a bit of troubleshooting info, as well as some tips and tricks. We will update this as we hear back from users about things they have run in to.

## ETCD

```
I lost an etcd pod and now it won't start up. The logs say "Was the raft log corrupted, truncated, or lost?".
```

First off, fear not, there are multiple ways to recover from this. If it is only a minority of etcd pods that were lost (1/3 or 2/5) then the fix is very simple. You need to remove and re-add the member to let etcd know that it is ok to recover that member. As an example:

```
#etcd-1 was lost. etcd-0 and etcd-2 are healthy
kubectl exec -it etcd-0 /bin/bash
export ETCDCTL_API=3
etcdctl member list
63cdb3774b98cc2e, started, etcd-2, http://etcd-2.etcd:2380, http://etcd-2.etcd:2379
69691ed70da97612, started, etcd-0, http://etcd-0.etcd:2380, http://etcd-0.etcd:2379
7e44220b60d58d6a, started, etcd-1, http://etcd-1.etcd:2380, http://etcd-1.etcd:2379
# we want to remove the last one
etcdctl member remove 7e44220b60d58d6a 
Member 7e44220b60d58d6a removed from cluster 9c3a8eb86bab5e36
# now re-add it
etcdctl member add etcd-1 --peer-urls http://etcd-1.etcd:2380
```

The pod should now automatically start ok on the next try (it was likely in crashloopbackoff).

If you have lost a majority of etcd nodes, please immediately grab your etcd backups and store them in a safe place:

```
$ kubectl get pods | grep "etcd-backup"
etcd-backup-3266148700-fg6c4   1/1       Running   0          14h
$ kubectl cp etcd-backup-3266148700-fg6c4:/srv/persist etcd_backups
#your backups are now in the etcd_backups directory
```

Then contact btrdb@googlegroups.com for help, or adapt the instructions for [etcd disaster recovery](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/recovery.md).

The reason this happens is that in the current versions of smartgridstore we do not attach real persistent storage to the pods. This is probably temporary and we may change this, but it is because our initial testing showed that RBDs did not perform well enough under the synchronous load of etcd.
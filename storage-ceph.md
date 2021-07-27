Rather than NFS, [Rook](https://rook.io) offers nearly turn key storage options.

Multiple storage engines are offered. We are interested in Ceph, as it can provide
block storage to pods via PersistentVolumeClaims.

Required are 3 or more hosts with SSDs, all part of the kube cluster. Ideally you'll
want these kube hosts to have a label and taint so that the ceph resources only run
on the storage nodes, and the storage nodes only run ceph.

In this example, I have a 3 node cluster, all of which are master control plane nodes.
I'll be adding further compute nodes to the cluster later.

This is based on the [official quickstart guide](https://rook.io/docs/rook/v1.6/ceph-quickstart.html),
with notes on what to change to work for my situation.

```
git clone https://github.com/rook/rook.git
cd cluster/examples/kubernetes/ceph
```

Start by spinning up the operator. If you don't have any non-master nodes in your cluster,
make this change to the operator.yaml file and then apply.

```

# for the deployment at the bottom of operator.yaml, set spec.template.spec.tolerations to:
# [{"key": "node-role.kubernetes.io/master", "effect": "NoSchedule", "operator": "Exists"}]
#
# Search the file for `CSI_PROVISIONER_TOLERATIONS` and uncomment, set it like this:
#  CSI_PROVISIONER_TOLERATIONS: |
#    [{"key": "node-role.kubernetes.io/master", "effect": "NoSchedule", "operator": "Exists"}]
# Do the same for CSI_PLUGIN_TOLERATIONS

kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

Once the operator pod is running successfully, you can customise the `cluster.yaml` file and apply.
`cluster.yaml` contains a single CR.
```
# spec.placement:
#    all:
#       tolerations: [{"key": "node-role.kubernetes.io/master", "effect": "NoSchedule", "operator": "Exists"}]

# storage.useAllNodes: false
# storage.useAllDevices: false

# storage.nodes:
# - name: kubemaster1
#   devices: [ {"name": "sdb" }]
# (there's lots of ways to configure which storage devices to use, this is the simplest)

kubectl create -f cluster.yaml
```

It can take some minutes for each of the successive pods to pull and execute. It may look like it's doing
nothing for a while.

You should eventually see multiple pods each of rook-ceph-mon, rook-ceph-mgr, and rook-ceph-osd. Note that
the osd-prepare pods are different. If the osd pods are not present, but the osd-prepare ones are, check the
logs on the osd-prepare pods.

When it is finished, the `ceph status` command via the toolbox should show 3 daemons, a mgr, and 3 osds. The
available should show the size of your total block devices allocated.

Ceph is now healthy.

Blindly apply the storage class and ceph block pool resources from this repo:
```
kubectl apply -f storage-ceph-storageclass.yaml
```

And hey presto, you should be able to provision storage:

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  storageClassName: rook-ceph-block
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
EOF
kubectl get pv
# NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS      REASON   AGE
# pvc-4a6ec218-6b3f-432b-b760-0fcdd0285ed3   20Gi       RWO            Delete           Bound    default/mysql-pv-claim   rook-ceph-block            57s

```


Problems I ran in to and how to debug them...

The first bunch of problems I had was because I hadn't set up my network correctly. Only monitor
pods were running. I had neglected to allow ip forwarding on all of my nodes, so docker's default
ip tables forward rule of drop was being applied.

I had re-created the cluster a few times. After deleting a cluster, you need to delete `/var/lib/rook`
from each node (this path is set in `spec.dataDirHostPath` in `cluster.yaml`).

The status of the ceph cluster can be checked with:

```
kubectl  -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

It should show 3 mon daemons, at least 1 mgr, and 3 OSDs. I had 0 OSDs. This was because the disks
I had assigned still have LVM configuration from a previous ceph cluster, and ceph was cowardly
refusing to use them.

Example healthy output:

```
  cluster:
    id:     4de1433d-02e3-4c3a-843e-889d2deb8dc4
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 2m)
    mgr: a(active, since 64s)
    osd: 3 osds: 3 up (since 43s), 3 in (since 83s)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   15 MiB used, 335 GiB / 335 GiB avail
    pgs:     1 active+clean
```


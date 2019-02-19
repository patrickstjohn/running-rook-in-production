# Running Rook in Production
# Introduction
The purpose of this document is to share some lessons learned about deploying and operating a Rook/Ceph cluster in a production environment. Our team noticed that while there is solid documentation written by the Rook and Ceph teams there isn't much open information about running this sort of setup in a production environment. Hopefully, this helps and if you have lessons to share please contribute to this document.

Our team is currently running a cluster with Petabytes of storage and constant streaming data into the cluster at several Gigabytes per second. We have several workloads running on the cluster orchestrated by Kubernetes and Rook/Ceph has been able to accommodate the varying performance characteristics required by these different workloads.

You'll want to bookmark the following:
1.  http://docs.ceph.com/docs/master/
1.  https://rook.io/docs/rook/master/

# Deployment
We've noticed that in order to have a more stable production cluster you want to be explicit about several items prior to deployment. This way when you need to upgrade nodes and reboot them from time to time you don't run into multiple pieces of infrastructure on a single node or other undesirable  side effects. You can also set podAntiAffinity to achieve this as well as the newer allowMultiplePerNode configuration item.

*  Label nodes for various pieces of Ceph infrastructure (mon,mgr,etc.)

```yaml
cluster.yaml
  mon:
    count: 3
    allowMultiplePerNode: false
  placement:
    mon:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: role
              operator: In
              values:
              - mon-node
```
```
* Label a node that corresponds to the label you've specified in cluster.yaml
kubectl label node node1 role=mon-node
* View the labels
kubectl get nodes --show-labels
```

*  Set hostNetwork: true. You really don't want to OSDs drop out anytime a flannel pod restarts or if you need to upgrade your network overlay.

```yaml
cluster.yaml
  network:
    # toggle to use hostNetwork
    hostNetwork: true
```

*  Set your drive topology to account for separate metadata and data devices. You'll get an awful lot more performance from your hardware if you split that apart. In our case we have two devices: /dev/sdb for metadata and a larger drive, /dev/sdc for data. The drive /dev/sdb is a 3 disk raid5 cluster and /dev/sdc is a 21 disk raid6 cluster (each physical drive for both raid clusters is a 1.7 TB SAS drive, raids are hw raids using a single perc 730p mini controller card) . If your build supports it the recommendation is to have a SSD for the metadata/journal partition.

```yaml
cluster.yaml
  storage: # cluster level storage configuration and selection
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^sd[b-c]"
    metadataDevice: "sdb"
```

*  Automate tear-down and cleanup. You'll more than likely be building/rebuilding the cluster quite a few times as you try out different settings for your workload. We use ansible quite a bit for this but there are lots of ways to go about doing this. The two items we do on rebuild are: remove all previous state information from /var/lib/rook (dataDirHostPath) and clean the drives of any previous partitions using parted to clean out any partion tables UUID's or the like that Rook/Ceph might stumble on and think there is data they need to protect. Obviously, you'll want to alter this for your drive topology.

```
ansible node1,node2,node3 -m shell -a "rm -rf /var/lib/rook/*"

ansible node1,node2,node3 -m shell -a "for DRIVE in sdb sdc; do parted --script -a optimal /dev/\$DRIVE mklabel gpt mkpart primary 0% 100% && parted --script -a optimal /dev/\$DRIVE rm 1; done"
```

After you've done the above it's time to actually deploy Rook and Ceph on your Kubernetes cluster. Follow the docs at https://rook.io/docs/rook/master/quickstart.html in order to get up and running. Basically you'll be doing the following (read through the rest of the document and supporting documentation prior to deploying the Object Storage and Block Storage - object.yaml and storageclass.yaml respectively):

```
kubectl create -f operator.yaml
kubectl create -f cluster.yaml
kubectl create -f toolbox.yaml
kubectl create -f storageclass.yaml
kubectl create -f object.yaml
```

Once the operator, agents, discover pods are up and running in the rook-ceph-system namespace (`kubectl get pods -n rook-ceph-system -w`) and the OSD, mon, mgr, tools pods are running in the rook-ceph namespace (`kubectl get pods -n rook-ceph -w`) it's time to get a shell in the tools pod and look around.

## Manage Initial Setup and OSD Topology 
In order to get a shell up and running in a pod with proper settings and toolset to interact with the cluster run the following:

```
kubectl -n rook-ceph exec -it rook-ceph-tools -- /bin/bash
```

Assuming things went well you should be able to run `ceph -s` and health should indicate `HEALTH_OK` for the cluster. It's very likely that this might not be the case if some OSDs failed to properly format or if you haven't deployed a storageclass or object store.

If you see something like `osd: 120 osds: 119 up, 119 in` that means there is a problem with one of your OSDs that needs to be tracked down prior to deploying everything.

In order to troubleshoot look at the following commands:

```
ceph health detail
ceph osd tree
```

The `ceph osd tree` command will tell you what OSD are in, their size and what physical host they are running on. This will allow you to focus in on where you need to troubleshoot. The following commands will probably point you in the right direction (where uid is the osd pod corresponding to the physical host that's having issues):

```
kubectl get pods -n rook-ceph -o wide
kubectl logs rook-ceph-osd-<uid> -n rook-ceph
```

Sometimes while you are doing maintenance on a host you need to remove the OSDs from the pool and then take the host out of the cluster temporarily. In order to do that read the following:

1.  http://docs.ceph.com/docs/giant/rados/operations/add-or-rm-osds/
1.  https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/

Once you have all of your OSDs up and in and the cluster may still be in a WARNING state due to not enough Placement Groups. If after you've created your Block Storage (storageclass.yaml) and Object Storage (object.yaml) you still have this warning take a look at the following documents:

1.  http://docs.ceph.com/docs/mimic/rados/operations/placement-groups/
1.  https://ceph.com/pgcalc/

This will help you properly size your pools for the optimal number of placement groups. In order to see what your pools are run `ceph osd pool ls` and this command will return the currently configured pools in your cluster. It should look something like this:

```
my-store.rgw.control
my-store.rgw.meta
my-store.rgw.log
my-store.rgw.buckets.index
.rgw.root
my-store.rgw.buckets.data
replicapool
```

# Tuning

## Object Storage
At the time of writing this document Block Storage with Erasure Coded pools wasn't working. Additionally, while Rook defaults of ext4 as a drive format Ceph recommends xfs and so we went with that in our case (http://docs.ceph.com/docs/mimic/rados/configuration/storage-devices/). An example of our storageclass.yaml looks like this:

```yaml
apiVersion: ceph.rook.io/v1alpha1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 2
  # For an erasure-coded pool, comment out the replication size above and uncomment the following settings.
  # Make sure you have enough OSDs to support the replica size or erasure code chunks.
  # erasureCoded:
  #  dataChunks: 2
  #  codingChunks: 1
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: standard
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  # Specify the namespace of the rook cluster from which to create volumes.
  # If not specified, it will use `rook` as the default namespace of the cluster.
  # This is also the namespace where the cluster will be
  clusterNamespace: rook-ceph
  # Specify the filesystem type of the volume. If not specified, it will use `ext4`.
  fstype: xfs
```

*  Note: Once you have a fairly heavily loaded cluster (in our case over 1PB) deploying large persistent volumes (greater than 1TB) took several hours per volume. Once we changed from ext4 to xfs that problem went away and provisioning became fast again even on a loaded cluster.

## Block Storage
Block storage in Rook/Ceph works reasonably well out of the box. The only thing we deviated from on the initial deployment was switching from replication to Erasure Coding to save on some space. Our primary tuning was done in the RGW configuration to optimize storage and retrieval of objects.

In order to override the defaults you need to edit/apply settings to the `rook-config-override` ConfigMap in the rook-ceph namespace. Ours looks like this:

```yaml
apiVersion: v1
data:
  config: |
    [client.radosgw.gateway]
    rgw frontends = beast endpoint=0.0.0.0:80
    rgw dynamic resharding = false
    rgw max objs per shard = 300000
    [mon]
    mon_osd_cache_size = 500
    [osd]
    osd_scrub_begin_hour = 23
    osd_scrub_end_hour = 6
    [global]
    bluestore_block_db_size = 365072220160
    bluestore_block_wal_size = 365072220160
    bluestore_cache_kv_max = 1048576000
    rocksdb_cache_size = 1048576000
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook
```

This is where you'll set all your Ceph specific tuning parameters. Keep in mind that depending what you're tuning you'll have to restart the corresponding pod (ie. kubectl delete) in order for it to take the new settings. So in the case of the Rados Gateway settings you would have to restart all the rgw pods. Additionally, we scaled out the deployment of our Radows Gateway's from 1 to 8. Our object store has well over a hundred million objects and a PB of storage to give you an idea of scale.

For more details on how to create accounts to access and consume data from the object store look at this document from the Rook team: https://rook.io/docs/rook/master/object.html

## Ceph FileSystem
In our use-case we don't have much use for the Ceph FileSystem. Originally we using it for a monolithic store instead of the Object Storage. Once we got to several million files we had fairly severe stability issues. As a result we moved to the Object Store and stopped using the Ceph FileSystem.

# Monitoring
We're still pretty early days on monitoring of our cluster. The best places to look into this are likely via Prometheus, the integrated dashboard and commands like `ceph -w` on the rook tools pod.

1.  https://rook.io/docs/rook/master/monitoring.html
1.  https://rook.io/docs/rook/master/ceph-dashboard.html

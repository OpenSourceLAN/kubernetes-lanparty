NFS is a convenient way to attach portable storage to your containers. You can also do host-local storage, but this will leave the data on whatever host the pod runs on, instead of being available wherever the host runs.

This setup has one obvious drawback - your storage host is now a SPOF for all of your containers that use storage. So be careful when using this!

TODO: document a way to do some kind of HA storage :)

### Set up your server

```
apt-get install nfs-kernel-server
mkdir /games
chmod 777 /games
cat <<EOF > /etc/exports
/games 10.10.0.0/16(rw,sync,no_root_squash)
EOF
exportfs -a
```

If we export, say, `/games` on the server, we can have individual servers mount subfolders - so tron1 could mount `/games/tron1`. Being that it's NFS, containers can start on any host and access their data.

Since it's an almost-public NFS mount (restricted to the server subnet), we should probably do something like make it a zfs/btrfs mount that gets snapshotted every 5 minutes in case something does `rm -rf *` or a malicious user gets in

### Using storage in your pods

**Important:** Ensure nfs-common package is installed on all hosts. Your containers will fail to start if they cannot mount nfs, and nfs cannot be mounted without an nfs client.

Start by creating a StorageClass, PhysicalVolume and PhysicalVolumeClaim using [provided nfs-pv.yaml](configs/nfs-pv.yaml).

```
kubectl apply -f configs/nfs-pv.yaml
```

Then you can add the `volumes` and `volumeMounts` properties to your Pod spec:

```
    spec:
      containers:
      - name: tron
        image: squid:5000/armagetron
        env:
          - name: SERVER_NAME
            value: Test Server lolol222
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: nfs
          mountPath: "/games"
      volumes:
      - name: nfs
        persistentVolumeClaim:
          claimName: nfs

```

Note that the exported path of the PersistentVolume in `nfs-pv.yaml`  was `/games/tron`, but the NFS server exports `/games/` - we can mount subdirectories of an NFS export. This is useful for sharing a single export with multiple game servers.



# Velero-on-Openshift
![1_P-zj2MOE9-vh3QAJ6q1C6A](https://user-images.githubusercontent.com/38520491/157206518-8066a705-9291-4420-9279-83bc366174eb.png)

Velero is an open source tool to safely backup and restore, perform disaster recovery, and migrate Kubernetes cluster resources and persistent volumes.
In this document I describe how to setup velero on Openshift (OKD 4.7).

***

### Overview

It's the architect we are going to depoly:

![download](https://user-images.githubusercontent.com/38520491/157213404-c8fbfe52-0b19-44c0-973e-64ff4555c132.png)

Let's explain each component:

1- There is an external NFS server that we use as a storage to sore the backups. It's outside of OKD cluster and somewhere safe and isolated. It can be an Ubuntu VM with running NFS service on it.

2- The Minio deployment will claim it's on PV from that external NFS server.

3- The Restic daemonset which is running on every workers. It has some privileges to access every pod's volume. Restic will do the backing up job from every pod's PV.

4- Velero needs to get noticed about what volumes to backup. We do it by making some annotation on volume. The PVC watcher deployment is a monitoring application that watches every volume on the cluster. If it finds a volume without correct annotation, it will expose it in it's metrics. We will monitor PVC watcher metrics with prometheus. Also there will be a Grafana dashboard and alert for it.

5- The velero deployment is the core of this stack. We will use velero cli to manage creating and restoring backups.

***

### Setup NFS Server

I used an Ubuntu 20 with enough storage to store backups. process to install NFS server package with `apt` or some other tools.

`apt install nfs-kernel-server`

Create some directory for sharing with NFS.

The config file for NFS server is `/etc/exports`. Open it with desired text editor. Then add this line:

```"/exports/minio-storage" *(rw,sync,all_squash,anonuid=0,anongid=0) ```

Instead of `*` you can use your IP range of course or at least limit the access to this instance on firewall.

The reason behind using `anonuid=0` and `anongid=0` is that every file and directory that will be created in `/exports/minio-storage` gets the root ownerships, and in the future we won't face problem with connecting this storage to another Velero Stack. In some senario maybe we want to migrate our application to another cluster with another Velero. Or even in a disaster we loose production cluster and have to setup cluster and velero again. Every time you deploy a pod in Openshif or Kubernetes, it get a random UID and GID. And it gets complicated to access files on NFS. There is some ways to assige a specific UID and GIU to processes on a pod but I'm not gonna to that here.

***

### Setup Minio

Download the velero repo from GitHub. It includes velero binary command and some usefull yaml files.

```wget https://github.com/vmware-tanzu/velero/releases/download/v1.8.0/velero-v1.8.0-linux-amd64.tar.gz```

```tar -xzf velero-v1.2.0-linux-amd64.tar.gz```

Copy the velero binary command in PATH so you can use it globaly. Remember it usees credential just like `oc` or `kubectl` command. So you need to login before use velero command.

```cp velero-v1.8.0-linux-amd64/velero /usr/local/bin/```

The first setup of minio is without persistence, in order words, we are installing "as is" with emptyDir and default credentials (which is minio/minio123).

```
oc apply -f velero-v1.8.0-linux-amd64/examples/minio/00-minio-deployment.yaml
namespace/velero configured
deployment.apps/minio created
service/minio created
job.batch/minio-setup created
```

Check if the minio pod is running after the installation:

```
oc project velero
oc get pods
NAME                     READY     STATUS      RESTARTS   AGE
minio-6b8ff5c8b6-r5mf8   1/1       Running     0          45s
minio-setup-867lc        0/1       Completed   0          45s
```

We will need the PV and PVC objects to point to our NFS Server. The PVC object must be under “velero” project. Here is the example `minio-volume.yaml` file. You will need to change according to your NFS server:

```
---
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/bound-by-controller: "yes"
  finalizers:
  - kubernetes.io/pv-protection
  name: minio-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 15Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: minio-pv-claim
    namespace: velero
  nfs:
    path: /exports/minio-storage
    server: X.X.X.X
  persistentVolumeReclaimPolicy: Retain

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pv-claim
  namespace: velero
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 15Gi
  volumeName: minio-pv
```

```
oc apply -f minio-volume.yaml
persistentvolume/minio-pv created
persistentvolumeclaim/minio-pv-claim created
```

Before configuring minio to the PVC, make sure the PVC is bound to the PV (otherwise you will break your minio setup):

```
oc get pvc
NAME             STATUS    VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
minio-pv-claim   Bound     minio-pv   15Gi       RWO                           1m
```

This is the moment of truth. Let's configure minio `/storage` volume to use the pvc "minio-pv-claim" using the command "oc set". The minio pod is restarted and we can use `oc get pods -w` to watch it:

```
oc set volume deployment.apps/minio --add --name=storage -t pvc --claim-name=minio-pv-claim --overwrite
deployment.apps/minio volume updated

oc get pods -w
NAME                     READY     STATUS              RESTARTS  AGE
minio-6b8ff5c8b6-r5mf8   0/1       Terminating         0         3m
minio-setup-867lc        0/1       Completed           0         3m
minio-6b8ff5c8b6-r5mf8   0/1       Terminating         0         3m
minio-6b8ff5c8b6-r5mf8   0/1       Terminating         0         3m
minio-68cbfb4c89-8rbjb   0/1       Pending             0         15s
minio-68cbfb4c89-8rbjb   0/1       Pending             0         15s
minio-68cbfb4c89-8rbjb   0/1       ContainerCreating   0         15s
minio-68cbfb4c89-8rbjb   1/1       Running             0         18s
```

Look up the ClusterIP and port of the minio service:

```
oc get services
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
minio     ClusterIP   10.144.222.38   <none>        9000/TCP   50s
```

Let's check if minio is responding to the port 9000. For such, we will ssh to one of the OKD master nodes (or any other node) run a curl to the ClusterIP:port for a response:

```
curl http://10.144.222.38:9000
<?xml version="1.0" encoding="UTF-8"?>
<Error><Code>AccessDenied</Code><Message>Access Denied.
(...)
```

Good. The minio service is responsive. Let's now download the standalone docker container with the minio client "mc". Remember, this is a standalone docker container and it does not translate service names, hence we need to use the actual ClusterIP. I personally use Podman instead and it's not important.

```
podman pull minio/mc
podman run -it --entrypoint=/bin/sh minio/mc
/ # mc config host add s3 http://10.144.222.38:9000 minio minio123
mc: Configuration written to `/root/.mc/config.json`. Please update your access credentials.
mc: Successfully created `/root/.mc/share`.
mc: Initialized share uploads `/root/.mc/share/uploads.json` file.
mc: Initialized share downloads `/root/.mc/share/downloads.json` file.
Added `s3` successfully.
```
We need to create the default "velero" bucket, otherwise, our installation of velero will fail.


```
/ # mc mb s3/velero
Bucket created successfully `s3/velero`.
/ # mc tree s3
s3
└─ velero
/ # mc ls s3
[2021-11-19 21:21:03 UTC]      0B velero/
```

Process to connect to NFS server, try to run `ls -la` command in `/exports/minio-storage` and see the contents that are created by Minio. We are ready to install velero with data persistence.

***

### Setup Velero

Velero supports backing up and restoring Kubernetes volumes using a free open-source backup tool called restic. This support is considered beta quality. Please see the list of limitations to understand if it fits your use case.

Restic is not tied to a specific storage platform, which means that this integration also paves the way for future work to enable cross-volume-type data migrations.

NOTE: hostPath volumes are not supported, but the local volume type is supported.

Restic needs to access the pod's volume on hosts so we need to grant access. So run the Restic pod in privileged mode. Add the velero ServiceAccount to the privileged SCC:

`oc adm policy add-scc-to-user privileged -z velero -n velero`

Create the credential file `credentials-velero`. The credentials must match the minio credentials you created before.

```
cat credentials-velero
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
```

To install Velero with Restic command, use the --use-restic flag. Remember we modify the Restic DaemonSet to request a privileged mode:

```
velero install \
      --provider aws \
      --plugins velero/velero-plugin-for-aws:v1.1.0 \
      --bucket velero \
      --secret-file ./credentials-velero \
	    --use-volume-snapshots=false \
      --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000 \
	    --use-restic && oc patch ds/restic --namespace velero --type json -p '[{"op":"add","path":"/spec/template/spec/containers/0/securityContext","value": { "privileged": true}}]'
      ```	  




















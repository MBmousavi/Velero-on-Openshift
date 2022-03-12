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

```tar -xzf velero-v1.8.0-linux-amd64.tar.gz```

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

We will need the PV and PVC objects to point to our NFS Server. The PVC object must be under “velero” project. Find it in `minio-volume.yaml` example file. You will need to modify it according to your NFS server. Run `oc apply -f minio-volume.yaml` to create it.

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

By default a userland openshift namespace will not schedule pods on all nodes in the cluster. To schedule on all nodes the namespace needs an annotation:

`oc annotate namespace velero openshift.io/node-selector=""`

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
      --use-restic
```

If Restic is not running in a privileged mode, it will not be able to access pods volumes within the mounted hostpath directory because of the default enforced SELinux mode configured in the host system level. You can create a custom SCC to relax the security in your cluster so that Restic pods are allowed to use the hostPath volume plug-in without granting them access to the privileged SCC.

```
oc patch ds/restic --namespace velero --type json -p '[{"op":"add","path":"/spec/template/spec/containers/0/securityContext","value": { "privileged": true}}]'
```	  

You can run "kubectl logs" to check the velero installation. If you do not see errors is because it installed correctly.

`kubectl logs deployment/velero -n velero`

***
### Setup Monitoring and PVC Watcher

velero-pvc-watcher is a Prometheus exporter that monitores all PVCs in the cluster and verifies that a matching `backup.velero.io/backup-volumes` or a `backup.velero.io/backup-volumes-excludes` is set.

Note: Due to the design all unmounted PVCs will reported as not being backed up, since there is no configuration for them.

The PVC watcher comes with Helm. I'm going to download and install helm chart.

```
helm repo add bitsbeats https://bitsbeats.github.io/helm-charts/
helm repo update
helm pull bitsbeats/velero-pvc-watcher
tar xzf velero-pvc-watcher-0.2.4.tgz
helm install  velero-pvc-watcher ./velero-pvc-watcher --namespace velero
```

The PVC metrics will be exposed via ServiceName:2121/metrics and you can scrape it with prometheus. But if your monitoring stack is out side of Openshift cluster, you need to write a `Route` for creating a public url for PVC metrics.

An example of a Route for PVC Watcher, you can find in `route-pvc-watcher.yaml`. Run `oc apply -f route-pvc-watcher.yaml` to create the route. The metrics are available on `/metrics`.

Here is an example of config for prometheus to scaping it:

```
- job_name: 'velero-pvc-watcher'
  metrics_path: /metrics
  static_configs:
     - targets: ['velero-pvc-watcher.my-openshift.com']
```

Velero by default exposes it's metrics. We can create a `service` for exposing the metrics and scrap it with prometheus. Also it can be publicly exposed with a `Route`. Let's create a `service` for Velero, Find it in `velero-svc.yaml` and run it with `oc apply -f velero-svc.yaml`.

Now let's create a `Route` for it. Let's call it `route-velero.yaml`.Process to create it with `oc apply -f route-velero.yaml`.

Now everything is applied, Let's take a look of what we have in velero namespace:

```
oc get all -n velero
NAME                                      READY   STATUS      RESTARTS   AGE
pod/minio-7bbf447f64-4wccw                1/1     Running     0          5h51m
pod/minio-setup-2vc8d                     0/1     Completed   2          5h55m
pod/restic-2jntd                          1/1     Running     0          5h21m
pod/restic-2pf5j                          1/1     Running     0          5h9m
pod/restic-47khk                          1/1     Running     0          5h21m
pod/restic-4bh97                          1/1     Running     0          5h21m
pod/restic-dbbdz                          1/1     Running     0          5h21m
pod/restic-hl2r5                          1/1     Running     0          5h20m
pod/restic-sdlg9                          1/1     Running     0          5h21m
pod/restic-sm7ws                          1/1     Running     0          5h21m
pod/restic-t9z97                          1/1     Running     0          5h21m
pod/restic-vpplf                          1/1     Running     0          5h21m
pod/velero-c658f7549-fjl5j                1/1     Running     0          5h21m
pod/velero-pvc-watcher-84f6554745-hzhxg   1/1     Running     0          132m

NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/minio                ClusterIP   172.30.101.217   <none>        9000/TCP   5h55m
service/velero-pvc-watcher   ClusterIP   172.30.191.234   <none>        2121/TCP   132m
service/velero-svc           ClusterIP   172.30.109.43    <none>        80/TCP     24m

NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/restic   10        10        10      10           10          <none>          5h21m

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/minio                1/1     1            1           5h55m
deployment.apps/velero               1/1     1            1           5h21m
deployment.apps/velero-pvc-watcher   1/1     1            1           132m

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/minio-669589dc8c                0         0         0       5h55m
replicaset.apps/minio-7bbf447f64                1         1         1       5h51m
replicaset.apps/velero-c658f7549                1         1         1       5h21m
replicaset.apps/velero-pvc-watcher-84f6554745   1         1         1       132m

NAME                    COMPLETIONS   DURATION   AGE
job.batch/minio-setup   1/1           43s        5h55m

NAME                                                HOST/PORT                                PATH                 SERVICES             PORT
route.route.openshift.io/velero-pvc-watcher-route   velero-pvc-watcher.my-openshift.com      velero-pvc-watcher   http                 None
route.route.openshift.io/velero-svc-route           velero-svc.my-openshift.com              velero-svc           8085                 None

NAME             STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
minio-pv-claim   Bound    minio-pv   15Gi      RWO            standard-csi   5h53m
```

There is a Grafana dashboard that you can use for Velero, You can find it here: https://grafana.com/grafana/dashboards/11055

Refrences:
  
https://velero.io/docs/v1.8/restic/
  
https://github.com/bitsbeats/velero-pvc-watcher

***
### Velero in Action

Now Velero is up and running, We need to test a complete scenario: Create a backup, simulate a dicaster and restoring the backup.
As I mentioned before velero has a binary command that we use it to do all tasks. Run the `velero version` to see the current server and client installed version:
  
```
Version: v1.8.0
      Git commit: a818c97dde2034385d7bf25e0ac2a2e055a0b372
Server:
      Version: v1.8.0
```

You will get an Unauthorized error in the Server section if you forget to `oc login` before that.
  
For testing scenarion I deploy a Nginx with PV and PVC in a namespace, and create a backup of it with velero, then I delete the whole namespace and I will try to restore the backup.

Let's create a test namespace and grant proper service acounts.

```
oc create namespace mousavi
oc create serviceaccount runasanyuid -n mousavi
```

Deploy the nginx with PVC with `oc apply -f nginx-pvc.yaml`

I put some random static contents in nginx root directory (/usr/share/nginx/html) which is mounted in it's PV.
  
Now let's get to backup. I'm gonna create a backup in "namespace level". It means backup every resources in namespace. 
  
```
velero backup create nginx-pvc --include-namespaces mousavi --wait

Backup request "nginx-pvc" submitted successfully.
Waiting for backup to complete. You may safely press ctrl-c to stop waiting - your backup will continue in the background.
...................................
Backup completed with status: Completed. You may check for more information using the commands `velero backup describe nginx-pvc` and `velero backup logs nginx-pvc`.
```

Ok, The backup is done, Let's check the backup list.
  
```
velero get backup
  
NAME        STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
nginx-pvc   Completed   0        0          2022-03-12 10:14:51 -0500 EST   29d       default            <none>
```
Now let's the namespace, It will delete all the resources in the namespace, PV, serviceaccount, service, route, ...
  
```
oc delete project mousavi
```

Now restore the backup:

```
velero restore create --from-backup nginx-pvc --wait
```

After a few moments you can see everything in the namespace have been created again.

Check the restore status:

```
velero get restore

NAME                       BACKUP      STATUS      STARTED                         COMPLETED                       ERRORS   WARNINGS   CREATED                         SELECTOR
nginx-pvc-20220312102836   nginx-pvc   Completed   2022-03-12 10:28:36 -0500 EST   2022-03-12 11:07:00 -0500 EST   0        0          2022-03-12 10:28:36 -0500 EST   <none>
```

We can create scheduled backup with velero, for example run a backup job every houre or every night. You can find the complete document here: https://docs.d2iq.com/dkp/konvoy/1.6/backup/

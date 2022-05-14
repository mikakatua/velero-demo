# Velero Backup & Restore
Requirements: [Minikube cluster using the docker driver](https://minikube.sigs.k8s.io/docs/drivers/docker/)

## MinIO setup
Run a container with MinIO (in the same network as minikube):
```
docker run -d \
  --name=minio \
  --net=minikube \
  -p 9000:9000 \
  -p 9001:9001 \
  minio/minio server /data --console-address ":9001"
```
Default credentials are minioadmin:minioadmin

Log in to http://localhost:9001/buckets and create a bucket named `velero-bucket`

## Install a Local Volume Provisioner
We want to demonstrate how to backup/restore persistent volumes using [restic](https://github.com/restic/restic). By default, minikube provides PVs of type `hostPath` which are not supported by restic.

For this example we will use [local persistent volumes](https://kubernetes.io/docs/concepts/storage/volumes/#local) deploying a [local volume static provisioner](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) in minikube.

It is worth to mention that it is also possible to use an `emptyDir` volume instead of a PV.

Log in to minikube with `minikube ssh` and run the following to prepare the storage:
```
sudo -i
for vol in vol{1..3}
do
  mkdir -p /mnt/fast-disks/$vol
  mount -t tmpfs $vol /mnt/fast-disks/$vol
done
```

Deploy the provisioner
```
kubectl create -n kube-system -f local-volume-provisioner.yaml
```

## Deploy the example
This will create the namespace `velero-example` with a Pod which mounts a local volume in the `/target` directory
```
kubectl create -f pod-with-pvc.yaml
```
Create some demo data and verify that it is persisted in the local volume
```
kubectl exec -n velero-example example-pod -- \
  sh -c 'echo "VERY IMPORTANT DATA" > /target/file.txt'
minikube ssh -- find /mnt/fast-disks -type f
```

## Velero setup
Create a file with the MinIO credentials:
```
cat <<! > credentials-minio
[default]
aws_access_key_id = minioadmin
aws_secret_access_key = minioadmin
!
```

Install velero with restic support:
```
MINIO_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' minio)
MINIO_BUCKET=velero-bucket

velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.4.0 \
  --bucket $MINIO_BUCKET \
  --secret-file ./credentials-minio \
  --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://$MINIO_IP:9000 \
  --use-volume-snapshots=false \
  --default-volumes-to-restic \
  --use-restic
```

## Backup and restore
To backup the data, restic needs to know the Pod which mounts the volume. It will use the directory `/var/lib/kubelet/pods` of the node to access the pod volume data. Velero’s restic integration can only backup volumes that are mounted by a pod and not directly from the PVC. The same applies to the restore operation, the Pod must also be restored.

For the backup to work using the [opt-in pod volume backup approach](https://velero.io/docs/v1.8/restic/#using-opt-in-pod-volume-backup), it is mandatory to add an annotation to the Pod with the name of the volume. Velero uses that annotation to discover Pod volumes that needs to be backed up. For example:
```
kubectl annotate pod/example-pod backup.velero.io/backup-volumes=data
```

Backing up the Pod using a label as a selector:
```
velero backup create backup1 --include-namespaces velero-example --selector run=busybox
velero backup describe backup1 --details
```

Delete the namespace (the associated PV will be released and its contents deleted):
```
kubectl delete ns velero-example
```

Restore the Pod (will create also the namespace):
```
velero restore create --from-backup backup1
```
Check that the data has been recovered:
```
minikube ssh -- find /mnt/fast-disks -type f
```

Other restore options include:
* Partial restoration (with the `--include-resources` option)
* Restore to a different namespace (with the `--namespace-mappings` option)
* Migration across different clusters. See [Cluster migration](https://velero.io/docs/v1.8/migration-case/)

## TODO
Investigate velero CSI support using [minikube CSI driver](https://minikube.sigs.k8s.io/docs/tutorials/volume_snapshots_and_csi/)
* [Beta Container Storage Interface Snapshot Support in Velero](https://velero.io/docs/main/csi/)
* [CSI Snapshotter](https://github.com/kubernetes-csi/external-snapshotter)
* [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
* [MySQL backups (with fsfreeze)](https://docs.ondat.io/docs/usecases/velero-backups/)
```
#  --features=EnableCSI \
#  --plugins velero/velero-plugin-for-aws:v1.4.0,velero/velero-plugin-for-csi:v0.2.0 \
#  --use-volume-snapshots=true \
#  --snapshot-location-config region=minio \
```

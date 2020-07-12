# Test using Velero Trident Recovery

The point of this test is to see if we can recover Trident Configurations using Velero Backup solution.

curl https://github.com/vmware-tanzu/velero/releases/download/v1.4.0/velero-v1.4.0-linux-amd64.tar.gz --output velero-v1.4.0-linux-amd64.tar.gz


## Setup

1. Velero needs an S3 store. We are going to use minio as that s3 store.
   1. From `vaultweb1.cloud.wwtatc.local` start a minio container:

        ```sh
        [root@vaultweb1 ~]# docker run -d -p 9000:9000   --name minio1   -v /data:/data   -e "MINIO_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE"   -e "MINIO_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"   minio/minio server /data
        ```
   2. Log into the minio interface and create the bucket `velero`
2. From `vaultocp3a` run the following

```sh
cat << EOF > /tmp/velero-creds
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF
velero install \
     --provider aws \
     --plugins velero/velero-plugin-for-aws:v1.0.0 \
     --bucket velero \
     --secret-file /tmp/velero-creds \
     --use-volume-snapshots=false \
     --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://vaultweb1.cloud.wwtatc.local:9000
```


3. Create your first backup with velero to backup the Trident namespace and statefultesting namespace that has our application.
```sh
velero backup create testbackup --include-namespaces="trident,statefultesting"  --include-cluster-resources
velero backup describe testbackup
velero backup get
```

4. Install Velero on `vaultocp3b` [ref](https://velero.io/docs/master/basic-install/)

```sh
cat << EOF > /tmp/velero-creds
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF
velero install \
     --provider aws \
     --plugins velero/velero-plugin-for-aws:v1.0.0 \
     --bucket velero \
     --secret-file /tmp/velero-creds \
     --use-volume-snapshots=false \
     --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://vaultweb1.cloud.wwtatc.local:9000
```

5. Verify you can see the backups on `vaultocp3b` by issuing: `velero get backups`

6. shutdown `vaultocp3a`

7. Restore Trident and PV's first

```sh
velero restore create --from-backup testbackup --include-resources PersistentVolume
velero restore create --from-backup testbackup --include-namespaces trident --include-cluster-resources
```
8. If Trident was already running, restart Trident to pick up the recovered volumes

```
tridentctl get volumes -n trident
kubectl scale --replicas=0 deployment/trident -n trident
sleep 5
kubectl scale --replicas=1 deployment/trident -n trident
tridentctl get volumes -n trident
```

9.  Verify You can see the persistent volumes:
```sh
oc get pv
```

9. Restore our application
```sh
velero restore create --from-backup testbackup --include-namespaces statefultesting
```

10. In testing it seemed like the NAS pod came up quickly, but it took a while (5 min) for the SAN pod to mount the storage.

11. If Trident was already running on `vaultocp3b` before the restore, it doesn't seem to pick up the new volumes until you restart it.

## Notables

* This process assumes that the destination openshift cluster is already part of the igroups are already provisioned. Two possible options are:
    * pre-load the initiators on the production side, so that there doesn't need to be any additional steps on the destination cluster.
    * run automation to load up the initiators and export policies before running through the restore process.
* If you are utilizing secrets to store your sensitive data, you will need to be extra cautious around the security of minio. This is because the secret object type in k8s is just base64 encoded.  All resources defined in the namespaces including secrets would be stored in compressed format on the minio server.


# Test Import of Single Volume

This test will utilize the Trident Import feature to import an existing volume.

1. Grab the trident volume data from the `vaultocp3a` and copy the yaml to `vaultocp3b`. (This is just really a lazy factor so we don't have to remember the volume name)
    ```
    oc get pvc/www-web-nas-0 -o yaml > web-nas-pvc.yaml
    oc get pvc/www-web-san-0 -o yaml > web-san-pvc.yaml
    oc get -o yaml tridentvolumes $(cat web-nas-pvc.yaml  | grep volumeName | awk '{print $2}') -n trident | grep internalName | awk '{print $2}' > trident-volume-name-nas.txt
    oc get -o yaml tridentvolumes $(cat web-san-pvc.yaml  | grep volumeName | awk '{print $2}') -n trident | grep internalName | awk '{print $2}' > trident-volume-name-san.txt
    scp trident-volume-name-san.txt vaultocp3b:/tmp/
    scp trident-volume-name-nas.txt vaultocp3b:/tmp/
    ```

2. Shutdown `vaultocp3a`

3. Import the volume using trident on `vaultocp3b`
    ```
    cat << EOF > /tmp/import_this_pvc_nas.yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: www-web-nas-0
      namespace: statefultesting
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: ontap-nas
    EOF
    tridentctl import volume cvnas $(cat /tmp/trident-volume-name-nas.txt) -f /tmp/import_this_pvc_nas.yaml -n trident --no-manage
    cat << EOF > /tmp/import_this_pvc_san.yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: www-web-san-0
      namespace: statefultesting
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: ontap-san
    EOF
    tridentctl import volume cvsan $(cat /tmp/trident-volume-name-san.txt) -f /tmp/import_this_pvc_san.yaml -n trident --no-manage
    ```

4. Delete the PVC that was just created

    ```
    oc delete pvc/www-web-nas-0
    oc delete pvc/www-web-san-0
    ```

5. Add the application using `oc apply -f <filename>` from [stateful_application.yaml](./stateful_application.yaml)

## Notables

1. The import process renames the volume. This may be an issue when:
   1. SVM-DR - Would it not see the volume and create a new volume? If it still functioned, would reverse replication cause an issue in a production recovery?
   2. Troubleshooting - The volume no longer matchs the junction-path. 
   3. Automation - We would potentially need to keep track of changes as we perform application recovery testing, and keep some table map between production and vault.
2. In the testing, where we were using the same SVM, and not testing with SVM-DR this orphaned the production trident configuration. Metadata in the Production Trident was still refering to the old name.  This meant everytime I needed to test this scenario (without SVM-DR) I had to redeploy the application in "production"
3. `ontap-san` is not supported. Thus iSCSI PV's will not work with this operation.
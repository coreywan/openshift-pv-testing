# Persistent Openshift Application DR

This is repo is just a way to document our findings around exploring Application DR for persistent workloads in Openshift.  Specifically, our exploration is using Openshift 3.11, and Netapp Ontap with Trident as the storage orchestration.

## Architecture

1. "Production" Openshift cluster: vaultocp3a.cloud.wwtatc.local [Inventory File](./ocp3a_inventory)
2. "Recovery" Openshift cluster: vaultocp3b.cloud.wwtatc.local [Inventory File](./ocp3b_inventory)
3. Netapp Ontap Cluster: na-ontap-cl1.cloud.wwtatc.local

The Openshift clusters were deployed using [AIO](https://www.openshift.com/blog/openshift-all-in-one-aio-for-labs-and-fun) (All-in-One) 


## Deployment

1. Deploy RHEL 7 hosts for the clusters (vaultocp3a.cloud.wwtatc.local, and vaultocp3b.cloud.wwtatc.local) on [vcsa1.cloud.wwtatc.local](https://vcsa1.cloud.wwtatc.local/ui)
   1. When Deploying, each VM was deployed with 8 CPUs, 16 GB RAM, and 100GB of a single disk.
2. Configure Each Host:

    ```
    subscription-manager register --org="{{ ORG ID }}" --activationkey="{{ ACTIVATION KEY }}" && yum update -y && reboot
    ssh-keygen
    ssh-copy-id $(hostname)
    subscription-manager attach --pool={{ OPENSHIFT POOL ID }}
    subscription-manager repos --disable="*"
    subscription-manager repos --enable="rhel-7-server-rpms" --enable="rhel-7-server-extras-rpms" --enable="rhel-7-server-ose-3.11-rpms" --enable="rhel-7-server-ansible-2.6-rpms"
    yum -y update
    yum -y install atomic-openshift-clients openshift-ansible
    ```

3. Copy over inventory to each host and execute the installation process. 

    * [ocp3a_inventory](./ocp3a_inventory)
    * [ocp3b_inventory](./ocp3b_inventory)

    ```
    ansible-playbook -i ocp3a_inventory /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
    ansible-playbook -i ocp3a_inventory /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml
    ```

4. Install Trident

    ```
    wget https://github.com/NetApp/trident/releases/download/v20.04.0/trident-installer-20.04.0.tar.gz
    tar -xf trident-installer-20.04.0.tar.gz
    cd trident-installer/
    cp tridentctl /usr/local/bin/
    tridentctl install -n trident
    ```

5. Copy over trident backend files to `vaultocp3a`
    * [cvnas](./backend-cvnas.json)
    * [cvsan](./backend-cvsan.json)

6. Apply trident backends on `vaultocp3a`
    ```
    tridentctl create backend -n trident -f ./backend-ontap-san.json 
    ```

7. Create Storage Classes on both clusters from [storage_classes.yaml](./storage_classes.yaml)

8. Create the Application on `vaultocp3a`. (Copy Over [stateful_application.yaml](./stateful_application.yaml))
    ```
    oc new-project statefultesting
    oc apply -f stateful_application.yaml
    ```

9.  Create the Project only on `vaultocp3b`.
    ```
    oc new-project statefultesting
    oc apply -f stateful_application.yaml
    ```

## Tests

1. [Import a Single Volume](./test_import_single_volume.md)
2. [Test using Velero Trident Recovery](./test_velero_backup_trident.md)
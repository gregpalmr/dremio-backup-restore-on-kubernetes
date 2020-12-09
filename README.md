# dremio-backup-restore-on-kubernetes
Dremio Data Lake Engine - Backup and Restore Procedure on Kubernetes

Use these instructions if you want to backup and restore Dremio meta-data on a Dremio Data Lake Engine running on a Kubernetes cluster.

### Prerequisites:

a. Launched a Dremio Data Lake Engine instance on a Kubernetes cluster using the Dremio Helm chart located at: 

[https://github.com/dremio/dremio-cloud-tools/tree/master/charts/dremio_v2](https://github.com/dremio/dremio-cloud-tools/tree/master/charts/dremio_v2)

b. The Dremio instance is running and in a healthy state.

c. Have a working **kubectl** cli session to your Kubernetes cluster.

## Step 1. Backup Dremio Coordinator node

Use this step to backup the Dremio meta-data from a running Dremio Data Lake Instance.

### Step 1.a. Run the Dremio backup command

Launch a bash terminal session into the Dremio Coordinator pod

     $ kubectl exec -it dremio-master-0 -- bash

Create a directory to place the backup files

     dremio-master-0:/opt/dremio $ mkdir -p /opt/dremio/data/backups

Execute the Dremio backup command

     dremio-master-0:/opt/dremio $ /opt/dremio/bin/dremio-admin backup -u <admin user> -p '<admin password>' -d /opt/dremio/data/backups

Create a single TAR file from the backup directory

     dremio-master-0:/opt/dremio $ tar cvzPf  \
          /opt/dremio/data/backups/dremio_backup_<date>.tar.gz \
          /opt/dremio/data/backups/dremio_backup_<date>

Exit the pod bash terminal session

     dremio@dremio-master-0:/opt/dremio $ exit

### Step 1.b. Copy the backup image to your local computer

Copy the TAR file from the pod to the local computer

     $ kubectl cp dremio-master-0:/opt/dremio/data/backups/dremio_backup_<date>.tar.gz dremio_backup_<date>.tar.gz

## Step 2. Restore Dremio meta-data to a new cluster.

Use this step to restore a Dremio backup to a new Dremio Data Lake Instance running on Kubernetes. Launch a new cluster using the Dremio Helm chart located at:

[https://github.com/dremio/dremio-cloud-tools/tree/master/charts/dremio_v2](https://github.com/dremio/dremio-cloud-tools/tree/master/charts/dremio_v2)

### Step 2.a. Start a Dremio admin pod to run offline commands

Use the helm chart command to stop the main Dremio pods and start a Dremio admin pod that attaches to the persistent volume claim.

     $ helm upgrade <dremio-cluster-name> dremio_v2 --reuse-values --set DremioAdmin=true

### Step 2.b. Copy the Dremio backup file to the admin pod

Remove the old backup files if they exists

     $ kubectl exec -it dremio-admin -- bash

     dremio-admin:/opt/dremio $ rm -rf /opt/dremio/data/backups

     dremio-admin:/opt/dremio $ mkdir -p /opt/dremio/data/backups

Exit the pod bash terminal session

     dremio-admin:/opt/dremio $ exit

Copy the backup file to the admin pod

     $ kubectl cp dremio_backup_<date>.tar.gz \
             dremio-admin:/opt/dremio/data/backups/dremio_backup_<date>.tar.gz 

### Step 2.c. Restore the Dremio backup using the offline commands

Run the offline restore commands in the admin pod

     $ kubectl exec -it dremio-admin -- bash

mv the old Dremio meta-data directory

     dremio-admin:/opt/dremio $ mv /opt/dremio/data/db /opt/dremio/data/db.orig

     dremio-admin:/opt/dremio $ mkdir -p /opt/dremio/data/db

Extract the files from the TAR file

     dremio-admin:/opt/dremio $ tar xzvf dremio_backup_<date>.tar.gz -C /

Run the Dremio admin restore command

     dremio-admin:/opt/dremio $ /opt/dremio/bin/dremio-admin restore \
                                -d /opt/dremio/data/backups/dremio_backup_<date>

Exit the bash terminal session on the admin pod

     dremio-admin:/opt/dremio $ exit

### Step 2.d. Relaunch the main Dremio coordinator and executor pods

Stop the Dremio admin pod and start the Dremio pods

     $ helm upgrade  <dremio-cluster-name> dremio_v2 --reuse-values --set DremioAdmin=false

Wait for the pods to start and then get the Dremio coordinator service IP address

     $ kubectl get pods

     $ kubectl get service

Log into the the Dremio Web UI and see the restored objects

     http://<dremio coordinator service ip address>:9047

---

Direct questions and comments to greg@dremio.com


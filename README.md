# dremio-backup-restore-on-kubernetes
Dremio Data Lake Engine - Backup and Restore Procedure on Kubernetes

Use these instructions if you want to backup Dremio meta-data on a Dremio Data Lake Engine running on a Kubernetes cluster.

## Step 1. Backup Dremio Coordinator node

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

### Step 2. Copy the backup image to your desktop computer

Copy the TAR file from the pod to the local computer

     $ kubectl cp dremio-master-0:/opt/dremio/data/backups/dremio_backup_<date>.tar.gz dremio_backup_<date>.tar.gz

### Step 3. Start a Dremio Admin pod to run offline commands

Use the helm chart command to stop the main Dremio pods and start a "Dremio Admin" pod that attaches to the persistant volume claim.

     $ helm upgrade <dremio-cluster-name> dremio_v2 --reuse-values --set DremioAdmin=true

### Step 4. Copy the Dremio backup file to the admin pod


Remove the old backup files if they exists

     $ kubectl exec -it dremio-admin -- bash

     dremio-admin:/opt/dremio $ rm -rf /opt/dremio/data/backups

     dremio-admin:/opt/dremio $ mkdir -p /opt/dremio/data/backups

     dremio-admin:/opt/dremio $ exit

Copy the backup file to the admin pod

     $ kubectl cp dremio_backup_<date>.tar.gz \
             dremio-admin:/opt/dremio/data/backups/dremio_backup_<date>.tar.gz 

### Step 5. Restore the Dremio backup using the offline commands

Run the offline restore commands in the admin pod

     $ kubectl exec -it dremio-admin -- bash

mv the old Dremio meta-data directory

     dremio-admin:/opt/dremio $ mv /opt/dremio/data/db /opt/dremio/data/db.orig

     dremio-admin:/opt/dremio $ mkdir -p /opt/dremio/data/db

Extract the files from the TAR file

     dremio-admin:/opt/dremio $ cd /

     dremio-admin:/opt/dremio $ tar xzvf dremio_backup_<date>.tar.gz -C /

Run the Dremio admin restore command

     dremio-admin:/opt/dremio $ /opt/dremio/bin/dremio-admin restore \
          -d /opt/dremio/data/backups/dremio_backup_<date>

Exit the bash terminal session on the admin pod

     dremio-admin:/opt/dremio $ exit

### Step 6. Relaunch the main Dremio coordinator and executor pods

Stop the "Dremio Admin" pod and start the Dremio pods

     $ helm upgrade <release-name> dremio_v2 --reuse-values --set DremioAdmin=false

Wait for the pods to start and then get the Dremio coordinator service IP address

     $ kubectl get pods

     $ kubectl get service

Log into the the Dremio Web UI and see the restored objects

     http://<dremio coordinator service ip address>:9047

---

Direct questions and comments to greg@dremio.com


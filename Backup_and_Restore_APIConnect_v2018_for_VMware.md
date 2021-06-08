# Backup and Restore of API Connect v2018 for VMware

## Reference

1. IBM Knowledge Center https://www.ibm.com/support/knowledgecenter/SSMNED_2018/com.ibm.apic.install.doc/overview_backup_restore_vm.html

## 1.0. Understanding the constraints

**The following are extracted from Knowledge Center - with  comments.**

- Backups are intended for recovery of the Management, Portal, and Analytics subsystems onto the **same deployment** from which they were taken, or onto a **new replacement installation** in the same environment for **disaster recovery**. The same environment means the same network configuration and project directory as the original installation. Backups are not designed as a means for migrating an API Connect deployment from one environment to another.
  **Comment: We are using backup/restore for purpose of disaster recovery.** 

  - *The DNS host names of the public endpoints (that resolve to the load balancer) must be the **SAME**. E.g. cloud-admin-ui.think.org.*
  - *The DNS host names and the IP address of the individual servers can be different. In fact, for disaster recovery, it is fair to assume that the restore takes place in a secondary data center that have a different network space with different IP address, and host names*.

- You must back up both the Management and Portal subsystems at the same time, to ensure synchronicity across the services.

  ***Comment: Highly recommended to schedule the backup at the same time***

- If you have to perform a restore, you must complete the restoration of the Management Service first, and then immediately restore the Developer Portal. The backups of the Management and Portal must be taken at the same time to ensure that the Portal sites are consistent with Management database.

  **Comment: Recommend to minimally to backup/restore of the 2 subsystems - Management and Portal.** Analytics data is typically offloaded to a more permanent system like Splunk or external ELK. Gateway can be restore easily by backing up to DataPower domains, and the API Connect configuration will be pushed once the Gateway services is registered to the Management*

- When restoring, the Gateway and all deployed subsystems (Management, Analytics, and Developer Portal) must be at the same version level. 

- This document is written and tested for API Connect v2018.4.1.10.

## 2.0. Environment Context

The document describes the procedure to backup API Connect v2018 (version 2018.4.1.0) configuration and restore the configuration in an new deployment (assumed in different Data Center).  The following instructions asumes an API Connect deployment in `dev` mode.

Let say I have APICloud environment, with the following servers which forms the Green deployment. We are going to backup the configuration to a SFTP server.

1. green-management1.think.org - 172.16.102.111
2. green-analytics1.think.org - 172.16.102.112
3. green-portal1.think.org - 172.16.102.113

We are going to restore the configuration to an API Cloud consisting of the following servers which forms the Blue deployment.

1. blue-management1.think.org - 172.16.102.211
2. blue-analytics1.think.org - 172.16.102.212
3. blue-portal1.think.org - 172.16.102.213

## 3.0. Backup management subsystem

### 3.1. Configure Management subsystem for scheduled backup

For disaster recovery purposes, scheduled backup is recommended so that the backup time of management subsystem and the portal subsystem can be synchronized.

1. From the project folder (that contains the `apiconnect-up.yml`, configure backup for the management subsystem. You need to ensure the `user` has write access the `folder path`. You may modify the cron schedule that suits your requirement. Note: the `mgmt` is the name of the `SUBSYS_NAME`.

   1. SFTP backup

      ```sh
      $ apicup subsys set mgmt cassandra-backup-protocol sftp
      $ apicup subsys set mgmt cassandra-backup-path /backup
      $ apicup subsys set mgmt cassandra-backup-host 172.16.102.99
      $ apicup subsys set mgmt cassandra-backup-port 22
      $ apicup subsys set mgmt cassandra-backup-schedule "0 0 1 * *"
      $ apicup subsys set mgmt cassandra-backup-auth-user <sftpuser>
      $ apicup subsys set mgmt cassandra-backup-auth-pass <sftpuser-pass>
      ```

   2. Object Store backup

      ```sh
      $ apicup subsys set mgmt cassandra-backup-protocol objstore
      $ apicup subsys set mgmt cassandra-backup-path cloud-object-storage-7u-cos-standard-1si
      $ apicup subsys set mgmt cassandra-backup-host s3.au-syd.cloud-object-storage.appdomain.cloud/ad-syd
      $ apicup subsys set mgmt cassandra-backup-port 443
      $ apicup subsys set mgmt cassandra-backup-schedule "0 0 1 * *"
      $ apicup subsys set mgmt cassandra-backup-auth-user <s3user>
      $ apicup subsys set mgmt cassandra-backup-auth-pass <s3user-pass>
      ```

2. Run the command to activate / push the settings to the management subsystem; where `mgmt` is the name of the `SUBSYS_NAME`.

   ```sh
   $ apicup subsys install mgmt
   ```

3. Check the progress of the the backup by checking if the pod labeled `<cassandra cluster name>-backup` is running.

   ```sh
   $ ssh -l apicadm green-management1.think.org
   
   $ sudo -i
   # kubectl get po | grep backup
   ```

4. On backup completion, the backup file will be stored in the folder `/backup` in the backup server `172.16.102.99`.

   ```sh
   $ ls -l /backup
   
   -rw-r--r-- 1 khongks khongks 145701649 Apr 22 19:55 1587549284406812631-0-1.tar.gz
   ```

5. List the backups and the status; where `mgmt` is the name of the `SUBSYS_NAME`.

   ```sh
   $ apicup subsys exec mgmt list-backups
   
   Cluster                    Namespace   ID                    Timestamp                                 Status
   apiconnect-apiconnect-cc   default     1587549284406812631   2020-04-22 09:54:44.406812631 +0000 UTC   Complete
   ```

### 3.2. (optional) Executing on-demand backup

In most situation, taking regular scheduled backups and relying on those backups should be sufficient. However, if you need to take an ad-hoc backup, it is possible as well. We assume that you have install the backup parameters as above (with exception of the `cassadra-backup-schedule`).

1. To initate an on-demand backup, run the following command; where `mgmt` is the name of the `SUBSYS_NAME`.

   ``` sh
   $ apicup subsys exec mgmt backup
   
   Connection created
   Invoking request
   Backup ID:  1587549284406812631
   ```

2. List the backups and the status; where `mgmt` is the name of the `SUBSYS_NAME`.

   ```sh
   $ apicup subsys exec mgmt list-backups
   
   Cluster                    Namespace   ID                    Timestamp                                 Status
   apiconnect-apiconnect-cc   default     1587549284406812631   2020-04-22 09:54:44.406812631 +0000 UTC   Complete
   ```

## 4.0. Backup portal subsystem

### 4.1. Configure Portal subsystem for scheduled backup

1. From the project folder (that contains the `apiconnect-up.yml`, configure backup for the portal subsystem. You need to ensure the `user` has write access the `folder path`. You may modify the cron schedule that suits your requirement. Note: the `portal` is the name of the `SUBSYS_NAME`.

   ```sh
   $ apicup subsys set portal site-backup-host 172.16.102.99
   $ apicup subsys set portal site-backup-port 22
   $ apicup subsys set portal site-backup-auth-user <sftpuser>
   $ apicup subsys set portal site-backup-auth-pass <sftpuser-pass>
   $ apicup subsys set portal site-backup-path /site-backups
   $ apicup subsys set portal site-backup-protocol sftp
   $ apicup subsys set portal site-backup-schedule "0 0 1 * *"
   ```

2. Run the command to activate / push the settings to the portal subsystem; where `portal` is the name of the `SUBSYS_NAME`.

   ```sh
   $ apicup subsys install portal
   ```

3. On backup completion, the backup file will be stored in the folder `/site-backup` in the backup server `172.16.102.99`. Here, you can set the backups for the system configuration and for each sites.

   ```sh
   $ ls /site-backup
   
   -rw-rw-r-- 1 khongks khongks     3149 Apr 22 19:57 _portal_system_backup-20200422.095734.tar.gz
   -rw-r--r-- 1 khongks khongks 13036079 Apr 22 19:57 portal-web.think.org@e1@enterprise-20200422.095725.tar.gz
   -rw-r--r-- 1 khongks khongks 12097132 Apr 22 19:57 portal-web.think.org@thinker@test-20200422.095702.tar.gz
   -rw-r--r-- 1 khongks khongks 12904994 Apr 22 19:57 portal-web.think.org@think@uat-20200422.095715.tar.gz
   ```

4. List the backups; where `portal` is the name of the `SUBSYS_NAME`. 

   ```sh
   $ apicup subsys exec portal list-backups remote
   
   _portal_system_backup-20200422.095734.tar.gz
   portal-web.think.org@e1@enterprise-20200422.095725.tar.gz
   portal-web.think.org@think@uat-20200422.095715.tar.gz
   portal-web.think.org@thinker@test-20200422.095702.tar.gz
   ```

### 4.2. (optional) Executing on-demand backup

If you need to do an on-demand backup for the portal subsystem, it is important that you do this at the same time as the management subsystem. Therefore it is good to get this scripted.

1. To initiate an on-demand backup, run the following command; where `portal` is the name of the `SUBSYS_NAME`.  This command backup the entire system.

   ```sh
   $ apicup subsys exec portal backup
   ```

2. (optional) To backup just the system configuration (without the sites).

   ```sh
   $ apicup subsys exec portal backup-system
   ```

3. (optional) To backup all the sites.

   ```sh
   $ apicup subsys exec portal backup-site installed
   ```

4. (optional) To backup a particular site; where `UUID` is the site ID

   ```sh
   $ apicup subsys exec portal backup-site UUID
   ```

5. List the backups; where `portal` is the name of the `SUBSYS_NAME`. 

   ```sh
   $ apicup subsys exec portal list-backups remote
   
   _portal_system_backup-20200422.095734.tar.gz
   portal-web.think.org@e1@enterprise-20200422.095725.tar.gz
   portal-web.think.org@think@uat-20200422.095715.tar.gz
   portal-web.think.org@thinker@test-20200422.095702.tar.gz
   ```

## 5.0. Before you restore

Here, we are desribing a scenario where we are restoring the backups in a new DR site.

**The following are extracted from Knowledge Center - with  comments.**

- If you have to perform a restore, you must complete the restoration of the Management Service first, and then immediately restore the Developer Portal. The backups of the Management and Portal must be taken at the same time to ensure that the Portal sites are consistent with Management database.

- Restoring the Management Service requires database downtime and is a destructive process that deletes current data and copies backup data. During the restoration process, external traffic must be stopped.

- In a Disaster Recovery scenario, do not log in to the administration UI or attempt to configure or change any settings prior to restoring the backup. Restore the backup immediately after installing the subsystem.

  **Comment: The backup files must be shipped/copied to the DR site regularly.**

- To restore the management database, you must use the original project directory that was created with `apicup` during the initial product installation. You cannot restore the database without the initial project directory because it contains pertinent information about the cluster. The endpoints and certificates cannot change; the same endpoints and certificates will be used in the restored system. Note that successful restoration depends on use of a *single* `apicup` project for all subsystems, even those in a different cluster. Multiple projects will result in multiple certificate chains which will not match.

  **Comment: This is highly recommended that you also backup the project folder. Details in section 5.2**

- Map the DNS entries from the source cluster to the corresponding IP addresses on the target cluster. Record the DNS entries for each endpoint before starting the restore.

  **Comment: You must update the public endpoint DNS host names are updated to point to IP addresses of the load balancers in the DR site**. 

- When restoring the management database, the endpoints (on the new cluster which is the target for the restoration) have to be the same as those on the old cluster (the source of the backup). This includes all the endpoints for API Connect: api-manager-ui, cloud-admin-ui, consumer-api, platform-api; api-gateway, api-gw-service; analytics-ingestion, analytics-client; and portal-admin, portal-www.

  **Comment: While the public endpoints DNS host names MUST remain the same, it is expected that they will be resolved to different the IP address of the load balancers in the DR site.**

- When restoring, the Gateway and all deployed subsystems (Management, Analytics, and Developer Portal) must be at the same version level. 

  **Comment: Must ensure that the OVA image files and the apicup CLI used to generate the ISO files are of the same level.**

### 5.1. Copy backup files to DR site

1. Copy the backup files to the new SFTP locatlion

2. (optional) Ensure that the backup files are not corrupted

   ```sh
   tar -tzf <backup_file>
   ```

### 5.2. Generate new ISO files for OVA initialization in the DR site

1. Make a copy of the project folder. E.g. Old folder is `green-apic`, new foler is `blue-apic`.

   ```sh
   cp -pR green-apic blue-apic
   cd blue-apic
   ```

2. Update the interface (host name and IP address)  `apiconnect-up.yml` file the following parameters. Either you edit the file manually, or issue the command to first delete the old interface and create a new interface. If you have 3 servers, you need to ensure that you update all of them.

   ```sh
   apicup iface delete mgmt green-management1.think.org eth0
   apicup iface create mgmt blue-management1.think.org eth0 172.16.102.211/24 172.16.102.1
   
   apicup iface delete mgmt green-analytics1.think.org eth0
   apicup iface create mgmt blue-analytics1.think.org eth0 172.16.102.212/24 172.16.102.1
   
   apicup iface delete mgmt green-portal1.think.org eth0
   apicup iface create mgmt blue-portal1.think.org eth0 172.16.102.213/24 172.16.102.1
   ```

3. (optional) The backup files maybe copied into a different SFTP server; so you need to update the backup SFTP servers for both the management and portal subsystem.

   ```sh
   $ apicup subsys set mgmt cassandra-backup-protocol sftp
   $ apicup subsys set mgmt cassandra-backup-path /backup
   $ apicup subsys set mgmt cassandra-backup-host 172.16.102.199
   $ apicup subsys set mgmt cassandra-backup-port 22
   $ apicup subsys set mgmt cassandra-backup-schedule "0 0 1 * *"
   $ apicup subsys set mgmt cassandra-backup-auth-user <sftpuser>
   $ apicup subsys set mgmt cassandra-backup-auth-pass <sftpuser-pass>
   
   $ apicup subsys set portal site-backup-host 172.16.102.199
   $ apicup subsys set portal site-backup-port 22
   $ apicup subsys set portal site-backup-auth-user sftpuser
   $ apicup subsys set portal site-backup-auth-pass Passw0rd!
   $ apicup subsys set portal site-backup-path /site-backups
   $ apicup subsys set portal site-backup-protocol sftp
   $ apicup subsys set portal site-backup-schedule "0 0 1 * *"
   ```

4. Now, you need to delete all the output folder that holds the previous ISO files.

   ```sh
   $ rm -fr mgmtplan-out
   $ rm -fr analytplan-out
   $ rm -fr portalplan-out55
   ```

5. Finally, generate the new ISO files for all the subsystems.

   ```sh
   $ apicup subsys install mgmt --out mgmtplan-out
   $ apicup subsys install analyt --out analytplan-out
   $ apicup subsys install portal --out portalplan-out
   ```

### 5.3. Update the DNS

1. Update the DNS to ensure that all the public endpoint host names resolved to are IP address of the load balancer in the DR site.

### 5.4. Deploy the OVA files

1. Ensure that the number of servers in Primary and DR site are the name. If it is 3 in Primary Site and it must be 3 in DR Site.
2. Here, you must ensure that the OVA files used must be the same version.
3. Attached the new ISO files to each of the OVF deployment.
4. Power up the servers

### 5.5. Ensure all the pods in the subsystems are running

1. You must wait for all the Cassandra pods are up and running in the management subsystem.

   ```sh
   $ ssh -l apicadm blue-management1.think.org
   
   $ sudo -i
   $ apic status
   
   root@blue-management1:~# apic status
   INFO[0000] Log level: info
   
   Cluster members:
   - blue-management1.think.org (172.16.102.211)
     Version: 1.10.0
     Type: BOOTSTRAP_MASTER
     Install stage: DONE
     Upgrade stage: UPGRADE_DONE
     Subsystem type: management
     Subsystem version: 2018.4.1.10
     Subsystem detail: Done
     Reboot Required: false
     Docker status:
       Systemd unit: running
     Kubernetes status:
       Systemd unit: running
       Kubelet version: blue-management1 (4.4.0-174-generic) [Kubelet v1.16.6, Proxy v1.16.6]
     Etcd status: pod etcd-blue-management1 in namespace kube-system has status Running
     Addons: calico, dns, helm, kube-proxy, nginx-ingress,
   Etcd cluster state:
   - etcd member name: blue-management1.think.org, member id: 1080437116836464529, cluster id: 9803971764799682404, leader id: 1080437116836464529, revision: 125077, version: 3.3.17
   
   Pods Summary:
   NODE                    NAMESPACE          NAME                                                          READY        STATUS           REASON
   blue-management1        default            apiconnect-analytics-proxy-d5bd4d474-4lrxl                    1/1          Running
   blue-management1        default            apiconnect-apiconnect-cc-repair-1587603600-9hffx              0/1          Succeeded
   blue-management1        default            apiconnect-apiconnect-cc-sw79s                                1/1          Running
   blue-management1        default            apiconnect-apim-schema-init-job-7qcmq                         0/1          Succeeded
   blue-management1        default            apiconnect-apim-v2-694d467b88-k7dkm                           1/1          Running
   blue-management1        default            apiconnect-client-dl-srv-7f997fd958-tj5bc                     1/1          Running
   blue-management1        default            apiconnect-juhu-596b558c6b-v5n6d                              1/1          Running
   blue-management1        default            apiconnect-ldap-97dd65597-4fj2b                               1/1          Running
   blue-management1        default            apiconnect-lur-schema-init-job-s9zvw                          0/1          Succeeded
   blue-management1        default            apiconnect-lur-v2-f99f54fd-xr6rq                              1/1          Running
   blue-management1        default            apiconnect-ui-7869d95874-7d72w                                1/1          Running
   blue-management1        default            backup-9l878-lxfjf                                            0/1          Succeeded
   blue-management1        default            backup-hrt9s-qknck                                            0/1          Succeeded
   blue-management1        default            backup-jgjgv-gjsjk                                            0/1          Succeeded
   blue-management1        default            backup-sv8c8-lzqsq                                            0/1          Failed
   blue-management1        default            cassandra-operator-cassandra-operator-7665cc74cb-6qmtk        1/1          Running
   blue-management1        default            restore-c2vkp-8xspr                                           0/2          Succeeded
   blue-management1        kube-system        calico-kube-controllers-f664645d6-qwbt9                       1/1          Running
   blue-management1        kube-system        calico-node-r2w4p                                             1/1          Running
   blue-management1        kube-system        coredns-6c46b968f7-59rjd                                      1/1          Running
   blue-management1        kube-system        coredns-6c46b968f7-sq72t                                      1/1          Running
   blue-management1        kube-system        etcd-blue-management1                                         1/1          Running
   blue-management1        kube-system        ingress-nginx-ingress-controller-mp2r7                        1/1          Running
   blue-management1        kube-system        ingress-nginx-ingress-default-backend-7d66f58f5f-m5nzc        1/1          Running
   blue-management1        kube-system        kube-apiserver-blue-management1                               1/1          Running
   blue-management1        kube-system        kube-apiserver-proxy-blue-management1                         1/1          Running
   blue-management1        kube-system        kube-controller-manager-blue-management1                      1/1          Running
   blue-management1        kube-system        kube-proxy-l475q                                              1/1          Running
   blue-management1        kube-system        kube-scheduler-blue-management1                               1/1          Running
   blue-management1        kube-system        tiller-deploy-778d4df46b-b6sch                                1/1          Running
   ```

2. Similar, You must wait for all the pods are running in the portal subsystem.

   ```sh
   $ ssh -l apicadm blue-management1.think.org
   
   $ sudo -i
   $ apic status
   
   INFO[0000] Log level: info
   
   Cluster members:
   - blue-portal1.think.org (172.16.102.213)
     Version: 1.10.0
     Type: BOOTSTRAP_MASTER
     Install stage: DONE
     Upgrade stage: UPGRADE_DONE
     Subsystem type: portal
     Subsystem version: 2018.4.1.10
     Subsystem detail: Running helm install
     Reboot Required: false
     Docker status:
       Systemd unit: running
     Kubernetes status:
       Systemd unit: running
       Kubelet version: blue-portal1 (4.4.0-174-generic) [Kubelet v1.16.6, Proxy v1.16.6]
     Etcd status: pod etcd-blue-portal1 in namespace kube-system has status Running
     Addons: calico, dns, helm, kube-proxy, nginx-ingress,
   Etcd cluster state:
   - etcd member name: blue-portal1.think.org, member id: 6408290631454412138, cluster id: 11972802208578094454, leader id: 6408290631454412138, revision: 113467, version: 3.3.17
   
   Pods Summary:
   NODE                NAMESPACE          NAME                                                          READY        STATUS           REASON
   blue-portal1        default            apic-portal-apic-portal-db-xpglr                              2/2          Running
   blue-portal1        default            apic-portal-apic-portal-nginx-56b5dcf4d6-q4spg                1/1          Running
   blue-portal1        default            apic-portal-apic-portal-www-rcgwb                             2/2          Running
   blue-portal1        default            ptl-list-backups-cg5jm-nwfkz                                  0/1          Succeeded
   blue-portal1        default            ptl-list-sites-74xsq-b4zr5                                    0/1          Succeeded
   blue-portal1        default            ptl-list-sites-pvn8p-sss5s                                    0/1          Succeeded
   blue-portal1        default            ptl-list-sites-vdstq-xphkg                                    0/1          Succeeded
   blue-portal1        default            ptl-list-sites-xch6q-mzrjf                                    0/1          Succeeded
   blue-portal1        default            ptl-restore-all-j4qmf-lmhdc                                   0/1          Succeeded
   blue-portal1        default            ptl-restore-all-x5k9z-w64gp                                   0/1          Succeeded
   blue-portal1        kube-system        calico-kube-controllers-f664645d6-qmvvb                       1/1          Running
   blue-portal1        kube-system        calico-node-p2hhw                                             1/1          Running
   blue-portal1        kube-system        coredns-6c46b968f7-4md4f                                      1/1          Running
   blue-portal1        kube-system        coredns-6c46b968f7-sbw7g                                      1/1          Running
   blue-portal1        kube-system        etcd-blue-portal1                                             1/1          Running
   blue-portal1        kube-system        ingress-nginx-ingress-controller-d77zp                        1/1          Running
   blue-portal1        kube-system        ingress-nginx-ingress-default-backend-7d66f58f5f-hvs67        1/1          Running
   blue-portal1        kube-system        kube-apiserver-blue-portal1                                   1/1          Running
   blue-portal1        kube-system        kube-apiserver-proxy-blue-portal1                             1/1          Running
   blue-portal1        kube-system        kube-controller-manager-blue-portal1                          1/1          Running
   blue-portal1        kube-system        kube-proxy-dx7sm                                              1/1          Running
   blue-portal1        kube-system        kube-scheduler-blue-portal1                                   1/1          Running
   blue-portal1        kube-system        tiller-deploy-778d4df46b-z4r7g                                1/1          Running
   ```

### 5.6. Ensure there is enough space

1. Go into the Cassandra pod.

   ```sh
   $ kubectl exec -it <Cassandra-pod> --bash
   ```

2. Use the script to calculate the space available for restore.

   ```sh
   # current available space
   avail=$(df /var/db/ | awk 'NR==2{print $4}')
    
   #space occupied by /var/db/data/
   db_data=$(du -s /var/db/data/ | awk '{print$1}')
   				
   # Estimated available space after cleanup (avail + db_data) and a buffer of 15%
   total_space_avail=$(((($avail + $db_data) * 1024) * 85 / 100))
   echo $total_space_avail
   ```

- The value obtained in the above script is in bytes and must be calculated for every pod, and compared against `4x` the value, where `x` is the backup tar size of the corresponding backup file. 
- Each backup file is in format <backup-id>-<ordinal-of-cassandra pod>-<number-of-cassandra-pods in cluster>.tar.gz
- In the example script above, if the backup tar size of <backup-id>-0-3.tar.gz is 15*1024*1024 bytes, the value for `$total_space_avail` in Cassandra cc-0 pod must be around 60*1024*1024 bytes.
-  If free space is insufficient, create additional free space before starting the restore.

### 6.0. Restore management subsystem

1. In Disaster Recovery scenario (restoring on a new deployment), you will not be able to list-backup to find out the `backupID`. So you need to look at the file name in thre backup SFTP host. E.g. the file name is `1587549284406812631-0-1.tar.gz` means the `backupID` is `1587549284406812631`.

2. Restore the configuration of the management subsystem; where `mgmt` is the `SUBSYS_NAME` and `1587549284406812631` is the `backupID`.

   ```sh
   $ apicup subsys exec mgmt restore 1587549284406812631
   ```

3. Verify that the restore process completed successfully.

   - Examine the restore job and pod.

     ```sh
     # kubectl get jobs | grep restore
     restore-c2vkp                     1/1           5m49s      7m11s
     root@blue-management1:~
     
     # kubectl get pods | grep restore
     restore-c2vkp-8xspr                                      0/2     Completed   0          7m23s	
     ```

   - Check the logs, if completed you will see the following

     ```sh
     # kubectl describe pod restore-c2vkp-8xspr
     
     2020-04-22T14:44:44.896Z upgrade:upgrade Upgrade not needed for step 30 as the current schema version is 30 and target schema version is 30
     2020-04-22T14:44:44.896Z lur:server ======================================================================
     2020-04-22T14:44:44.896Z lur:server UPGRADE SCHEMA COMPLETE
     2020-04-22T14:44:44.896Z lur:server ======================================================================
     2020-04-22T14:44:44.896Z lur:server Schema upgraded and data populated, exiting...
     2020-04-22T14:44:44.897Z lur:server Not exiting since mode is restore_upgrade
     2020-04-22T14:44:45.070Z lur:server ======================================================================
     2020-04-22T14:44:45.070Z lur:server RESTORE START
     2020-04-22T14:44:45.070Z lur:server ======================================================================
     2020-04-22T14:44:45.070Z lur:server restoring since mode is restore_upgrade
     2020-04-22T14:44:45.070Z lur:server LUR restore is no-op right now
     2020-04-22T14:44:45.070Z lur:server ======================================================================
     2020-04-22T14:44:45.070Z lur:server RESTORE COMPLETE
     2020-04-22T14:44:45.070Z lur:server ======================================================================
     2020-04-22T14:44:45.070Z lur:server Restore finished, exiting...
     2020-04-22T14:44:51.899Z apim:server Bootstrapping is not performed in mode restore_upgrade
     2020-04-22T14:44:51.900Z apim:server ======================================================================
     2020-04-22T14:44:51.900Z apim:server RESTORE START
     2020-04-22T14:44:51.900Z apim:server ======================================================================
     2020-04-22T14:44:51.900Z apim:server restoring since mode is restore_upgrade
     2020-04-22T14:44:51.900Z apim:server Resyncing all gateway services
     2020-04-22T14:44:51.900Z apim:server ==============================
     2020-04-22T14:44:51.995Z apim:server - Checking gateway service: gateway-service (gateway-service)
     2020-04-22T14:44:52.603Z apim:server Gateway services resync complete
     2020-04-22T14:44:52.603Z apim:server ================================
     2020-04-22T14:44:52.603Z apim:server ======================================================================
     2020-04-22T14:44:52.603Z apim:server RESTORE COMPLETE
     2020-04-22T14:44:52.603Z apim:server ======================================================================
     2020-04-22T14:44:52.603Z apim:server Restore finished, exiting...
     ```

   - Watch the `ClusterRestoreStatus` inside `CassandraCluster` (cc) Custom Resource to see the current status of the Cassandra restore process. When compled, you will see the following:

     ```sh
     # kubectl get cc -o yaml | grep -A 1 ClusterRestoreStatus
         ClusterRestoreStatus: completed
         ConfiguredMode: daemonset
     
     # kubectl get cc -o yaml | grep -A 1 ClusterRestoreStatus
         ClusterRestoreStatus: completed
         ConfiguredMode: daemonset
     
     # kubectl get cc -o yaml | grep -A 1 ClusterRestoreStatus
         ClusterRestoreStatus: completed
         ConfiguredMode: daemonset
     ```

   4. Fix all stuck tasks:

      1. Download `apicops` from [https://github.com/ibm-apiconnect/apicops/releases](https://www.ibm.com/links?url=https%3A%2F%2Fgithub.com%2Fibm-apiconnect%2Fapicops%2Freleases). Login to one of the management server

         ```sh
         $ ssh -l apicadmin blue-management1.think.org
         $ sudo -i
         
         # wget https://github.com/ibm-apiconnect/apicops/releases/download/v0.2.86/apicops-linux
         # chmod +x apicops-linux
         # mv apicops-linux /usr/local/bin/apicops
         ```

      2. Remove any pending tasks.

         ```sh
         # apicops task-queue:fix-stuck-tasks
         
         Current date/time 2020-04-22T15:08:35.850Z
         apicops 0.2.86
         Apim Pod version is: 2018.4.1.10 ( apim:2018.4.1-1141-bd7c128 )
         No stuck send tasks found
         Deleted snapshot task id da5063ca-d7d8-48c3-a551-c247a6430e20 with payload containing webhook id a9e4852b-af76-4c06-9cf8-f8255622dfef as it did not exist in the webhook table
         No stuck snapshot tasks found
         No stuck heartbeat tasks found
         No stuck manage-stale-catalog-webhooks tasks found
         No stuck manage-stale-cloud-webhooks tasks found
         ```

      3. Check pending tasks, if any.

         ```sh
         # apicops task-queue:list-stuck-tasks
         
         Current date/time 2020-04-22T15:08:49.301Z
         apicops 0.2.86
         Apim Pod version is: 2018.4.1.10 ( apim:2018.4.1-1141-bd7c128 )
         No stuck send tasks found
         No stuck snapshot tasks found
         No stuck heartbeat tasks found
         No stuck manage-stale-catalog-webhooks tasks found
         No stuck manage-stale-cloud-webhooks tasks found
         ```

### 7.0. Restore portal subsystem

 Here, we are desribing a scenario where we are restoring the backups in a new DR site.

**The following are extracted from Knowledge Center - with  comments.**

- If you have to perform a restore, you must complete the restoration of the Management Service first, and then immediately restore the Developer Portal. The backups of the Management and Portal must be taken at the same time to ensure that the Portal sites are consistent with Management database.
- Restoration requires a functioning Developer Portal. In a disaster recovery scenario, you might need to reinstall the Developer Portal subsystem before you can restore the backed-up data.

1. List the system and site backups on the backup server; where `portal` is the `SUBSYS_NAME`.

   ```sh
   $ apicup subsys exec portal list-backups remote
   ```

2. (optional) To see what restore actions are performed (dry run) on the latest version of the backup. You can replace the word `now` to a timestamp in this format `YYYY-MM-DD HH:MM:SS` to retrieve the nearest backup to the specified time.

   ```sh
   $ apicup subsys exec portal restore-all dry now
   
   2020-04-22 16:24:17: looking for system and site backups to restore...
   2020-04-22 16:24:17: Looking for the latest system backup from local and remote filesystems from date: 20200422 16:24:16...
   2020-04-22 16:24:17: Fetching latest backup for _portal_system_backup* at 172.16.102.99:22:/site-backups from date: 20200422 16:24:16
   2020-04-22 16:24:17: Latest remote backup found as _portal_system_backup-20200422.095734.tar.gz
   2020-04-22 16:24:17: We found the latest backup as _portal_system_backup-20200422.095734.tar.gz
   2020-04-22 16:24:17: Found 3 site(s) to restore:
     portal-web.think.org/e1/enterprise
     portal-web.think.org/thinker/test
     portal-web.think.org/think/uat
   2020-04-22 16:24:17: Fetching latest backup for portal-web.think.org@e1@enterprise at 172.16.102.99:22:/site-backups from date: 20200422 16:24:16
   2020-04-22 16:24:18: Latest remote backup found as portal-web.think.org@e1@enterprise-20200422.095725.tar.gz
   2020-04-22 16:24:18: Downloading 172.16.102.99:22:/site-backups/portal-web.think.org@e1@enterprise-20200422.095725.tar.gz using sftp
   2020-04-22 16:24:19: Fetching latest backup for portal-web.think.org@thinker@test at 172.16.102.99:22:/site-backups from date: 20200422 16:24:16
   2020-04-22 16:24:20: Latest remote backup found as portal-web.think.org@thinker@test-20200422.095702.tar.gz
   2020-04-22 16:24:20: Downloading 172.16.102.99:22:/site-backups/portal-web.think.org@thinker@test-20200422.095702.tar.gz using sftp
   2020-04-22 16:24:21: Fetching latest backup for portal-web.think.org@think@uat at 172.16.102.99:22:/site-backups from date: 20200422 16:24:16
   2020-04-22 16:24:21: Latest remote backup found as portal-web.think.org@think@uat-20200422.095715.tar.gz
   2020-04-22 16:24:21: Downloading 172.16.102.99:22:/site-backups/portal-web.think.org@think@uat-20200422.095715.tar.gz using sftp
   2020-04-22 16:24:23: DRY_RUN: System backup will be restored from remote path: /site-backups/_portal_system_backup-20200422.095734.tar.gz
   DRY_RUN: Site will be restored. URL: portal-web.think.org/e1/enterprise, UUID: ff0a899a-59da-42f8-9fd5-fbe553f36960.91a08373-fceb-4161-8555-028bb4eae893, FILE: portal-web.think.org@e1@enterprise-20200422.095725.tar.gz
   DRY_RUN: Site will be restored. URL: portal-web.think.org/thinker/test, UUID: 615fa02a-3b30-4ff0-84b4-dce8b22eea63.8408b553-3f96-41cb-9acf-6e9d6a5a2659, FILE: portal-web.think.org@thinker@test-20200422.095702.tar.gz
   DRY_RUN: Site will be restored. URL: portal-web.think.org/think/uat, UUID: 615fa02a-3b30-4ff0-84b4-dce8b22eea63.bc0fda4e-627e-48a7-a440-38daeff9e741, FILE: portal-web.think.org@think@uat-20200422.095715.tar.gz
   ```

3. To restore to the latest version of the backup. You can replace the word `now` to a timestamp in this format `YYYY-MM-DD HH:MM:SS` to retrieve the nearest backup to the specified time. Note: This is not a `DRY RUN`.

   ```sh
   $ apicup subsys exec portal restore-all run now
   
   2020-04-22 16:25:39: looking for system and site backups to restore...
   2020-04-22 16:25:39: Looking for the latest system backup from local and remote filesystems from date: 20200422 16:25:39...
   2020-04-22 16:25:39: Fetching latest backup for _portal_system_backup* at 172.16.102.99:22:/site-backups from date: 20200422 16:25:39
   2020-04-22 16:25:40: Latest remote backup found as _portal_system_backup-20200422.095734.tar.gz
   2020-04-22 16:25:40: We found the latest backup as _portal_system_backup-20200422.095734.tar.gz
   2020-04-22 16:25:40: using system backup found remotely at: /site-backups/_portal_system_backup-20200422.095734.tar.gz
   2020-04-22 16:25:40: Downloading 172.16.102.99:22:/site-backups/_portal_system_backup-20200422.095734.tar.gz using sftp
   2020-04-22 16:25:41: The portal system configuration has been succesfully restored from _portal_system_backup-20200422.095734.tar.gz
   2020-04-22 16:25:42: Found 3 site(s) to restore:
     portal-web.think.org/e1/enterprise
     portal-web.think.org/thinker/test
     portal-web.think.org/think/uat
   2020-04-22 16:25:42: Latest local backup found as portal-web.think.org@e1@enterprise-20200422.095725.tar.gz
   2020-04-22 16:25:42: Fetching latest backup for portal-web.think.org@e1@enterprise at 172.16.102.99:22:/site-backups from date: 20200422 16:25:39
   2020-04-22 16:25:42: Latest remote backup found as portal-web.think.org@e1@enterprise-20200422.095725.tar.gz
   2020-04-22 16:25:43: Queueing restore for site: portal-web.think.org/e1/enterprise
   2020-04-22 16:25:44: Latest local backup found as portal-web.think.org@thinker@test-20200422.095702.tar.gz
   2020-04-22 16:25:44: Fetching latest backup for portal-web.think.org@thinker@test at 172.16.102.99:22:/site-backups from date: 20200422 16:25:39
   2020-04-22 16:25:44: Latest remote backup found as portal-web.think.org@thinker@test-20200422.095702.tar.gz
   2020-04-22 16:25:45: Queueing restore for site: portal-web.think.org/thinker/test
   2020-04-22 16:25:45: Latest local backup found as portal-web.think.org@think@uat-20200422.095715.tar.gz
   2020-04-22 16:25:45: Fetching latest backup for portal-web.think.org@think@uat at 172.16.102.99:22:/site-backups from date: 20200422 16:25:39
   2020-04-22 16:25:45: Latest remote backup found as portal-web.think.org@think@uat-20200422.095715.tar.gz
   2020-04-22 16:25:46: Queueing restore for site: portal-web.think.org/think/uat
   2020-04-22 16:25:47: Done. 3 / 3 queued for restoring
   ```

4. You can check the status of the restore. It should be completed when you see the portal sites are `INSTALLED`.

   ```sh
   $ apicup subsys exec portal list-sites sites
   
   615fa02a-3b30-4ff0-84b4-dce8b22eea63.bc0fda4e-627e-48a7-a440-38daeff9e741 => portal-web.think.org/think/uat (INSTALLED)
   ff0a899a-59da-42f8-9fd5-fbe553f36960.91a08373-fceb-4161-8555-028bb4eae893 => portal-web.think.org/e1/enterprise (INSTALLED)
   ```

   




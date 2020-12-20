# Implementation of Disaster Recovery For NFS
The Use of Distributed Replicated Block Device on two NFS server to prevent data loss

I will be implementing disaster recovery for two NFS servers using Distributed Replicated Block Device(DRBD) solution. When the primary NFS server fails,  the passive NFS server  becomes the primary server to prevent data loss.

I created and configured two NFS servers named nfs1 and nfs2 on two virtual machines.

Here are the steps I have followed.

 I added four  5GB disks to the machine.
On the first disk I created two partitions. 

I created a LVM type partition using 90% of the disk space by running the command ```gdisk /dev/sda -n``` and I changed the partition type to LVM by entering the 8e00 hex code for LVM. 

I created a second partition using the remaining space and gave it the swap partition type. I activated the swap partition by running ```mkswap /dev/sda2``` and ```swapon /dev/sda2```.

 I checked to see if my swap partition was created by running swapon -s. All was in order.

I created partitions on the remaining three disks using up all their spaces to create single partitions: ```gdisk /dev/sdb -n```, ```gdisk /dev/sdc -n``` and ```gdisk /dev/sdd -n```  and i made them LVM partitions.

I changed all four partitions into physical volumes using: pvcreate ```/dev/sda1 /dev/sdb1 /dev/sdc1 /dev/sdd1```. I used pvdisplay to check the physical volumes I made.

![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/pvcreate.jpg)

Using ```vgcreate vg_opt  /dev/sda1```, i turned my first physical volume into a volume group called vg_opt and created a logical volume using all available space: ```lvcreate -l 100%VG -n lv_propitix_opt vg_opt```.
Using ```vgcreate vg_apps /dev/sdb1``` i changed the second physical volume into a volume group. I then created a logical volume using all available space by running: ```lvcreate -l 100%VG -n lv_propitix_apps vg_apps```. I did the same for the third and fourth physical volumes. I made them into volume groups; ```vg_logs & vg_db``` and then into logical volumes: ```lv_propitix_logs & lv_propitix_db```. 

![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/lv.png)

 Then installed the nfs-kernel-server package by running ```apt install nfs-kernel-server```.

 I adjusted the permission settings of my ```/mnt/propitix_opt,/mnt/propitix_apps, /mnt/propitix_logs and /mnt/propitix_db``` folders granting read, write and execute access to all contents. I ran ```chown -R nobody:nogroup -directory``` and ```chmod 777 -directory```. I opened the ```/etc/exports``` file to add permissions for accessing the folders on the server. I added the /mnt folders one after the other into the file granting access to any client by following this order for all four folders ```/mnt/propitix_opt * (rw,sync,no_subtree_check)```. 
 ![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/chown.png)

then exported the fileshare directory by running ```exportfs -a and i``` restarted the NFS server by running ```systemctl restart nfs-kernel-server.
![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/export.png)

I ran allow ufw allow from any to any port nfs to grant access to the file share. Then I ran ufw status to check my configuration.

**Configuring NTP on NFS1 and NFS2 Servers.**

I ran apt update to update the server's packages and then ran ```apt install ntp``` to install the NTP server deamon.

Then Configured NTP Server by switching to an NTP server pool closest to your location
```sudo nano /etc/ntp.conf```
![](nfs1.png)

I enabled ntp to run at the system startup by running the command systemctl enable ntp
Then, open the ntp port 123 udp as follow:
```sudo ufw allow from any to any port 123 proto udp```

**Configuring DRBD on both NFS servers**

My DRBD Setup

- NFS server 1: has the IP address: 192.168.1.93; we refer to this one as nfs1.
- NFS server 2: has the IP address: 192.168.1.94; we refer to this one as nfs2.
The /data directory will be replicated by DRBD between nfs1 and nfs2. It will contain the NFS share /data.

I created the DRBD data partition that will be holding data to be shared between the NFS servers.
Created a  partition and formated it in xfs file system.
![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/diskf.png)

I created the ```/data``` directory that will be holding data to be shared across the subnet ```192.168.1.0/24``` using ```sudo mkdir -p /data```

With the directory  already created, i export it so that it can be accessible to the webserver  by  editing the file ```/etc/exports``` and added ```/data/	192.168.1.0/24(rw,sync,no_root_squash,no_subtree_check)``` at the end of the configuration. For the changes to take effect i restarted the nfs service.
![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/exp.png)

**Installation of DRBD**
I installed using ```sudo apt-get install drbd-utils```

After the installation of DRBD,i edited the config file located in /etc/drbd.conf. As shown below:

![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/config.png)

Configure  /etc/drbd.conf  file of nfs2 the same way as nfs1 shown above

Initialize the metadata storage on both nodes with ```sudo drbdadm create-md r0```

An error message  ```Command 'drbdmeta 0 v08 /dev/sdf1 internal create-md' terminated with exit code 40```  poped up ,and the error was clear with ```sudo dd if=/dev/zero of=/dev/sdf1 bs=1M count=128```

Then i retried the command ```sudo drbdadm create-md r0```
![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/metadata.png)
![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/metadata2.png)

Start the drbd service on both nodes by running the command ```sudo systemctl start drbd```

Confirm the drbd device is attached with ```df -h```
![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/confirm.png)

I Formatted the drbd device with xfs filesystem on both nodes as shown below
![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/format.png)

I Mounted the file system of the drbd device with ```sudo mount /dev/drbd0 /data```

I Created some files on nfs2 to proof evidence of data replication using ```sudo touch /data/file{1..5}```

Open the ```/data``` directory on Node 2 to check the contents
![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/check.png)

Open the ```/data``` directory on nfs1 to check the contents and its empty
![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/check2.png)

I Opened port 7788 on both nodes with ```sudo ufw allow 7788/tcp```

I Checked the DRBD Status with ```sudo cat /proc/drbd```
![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/cat.png)

I tested the DRBD Configuration by unmounting the volume drbd0 on  nfs 2 (```sudo umount /data ```) and then
 make nfs2 the secondary node(```sudo drbdadm secondary r0```) and nfs1 the primary node (```sudo drbdadm primary r0```), then check if the data(files created are available)

 Then i mounted the volume drbd0 on  nfs 1 with ```sudo mount /dev/drbd0 /data``` and then check if the files i created on nfs2 earlier were replicated as show below

 ![](https://github.com/drazen-dee28/Implement-Disaster-Recovery-For-NFS/blob/main/images/datanfs1.png)
 
 The 5 files created on nfs2(node 2)  are now replicated in nfs1(node 1) , hence our DRBD setup is successful

 Credits:

 [DRBD](https://www.globo.tech/learning-center/setup-drbd-9-ubuntu-16/)
 
 [Darey.io](https://learning.darey.io/)
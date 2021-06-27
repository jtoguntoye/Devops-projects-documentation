# Deploying web solution with WordPress

## Step 1- Preparing the web server
- Launch an EC2 instance to serve as the web server and added 3 EBS (Elastic Block Volume) in the same AZ( Availablilty Zone) to the EC2 instance

<img width="790" alt="ebs_volumes_server" src="https://user-images.githubusercontent.com/23315232/123272961-230e8a00-d4fa-11eb-87c1-e23f63f1200b.png">

- On terminal, check the block devices attached to the EC2 instance with command 
```
lsblk
```
<img width="404" alt="block_Devices_server" src="https://user-images.githubusercontent.com/23315232/123273247-649f3500-d4fa-11eb-8120-ca1228be06f7.png">

- Next, create a single partition on each of the three disks attached using the ``gdisk`` uitility
The newly configured partitions on the 3 disks are shwn below:

<img width="324" alt="blk_dev_after_gdisk_partitioning" src="https://user-images.githubusercontent.com/23315232/123274447-6f0dfe80-d4fb-11eb-8add-66252a6d1aeb.png">

- Install lvm2 package to be used to create logical volumes with the command:
```
sudo yum install lvm2
```
 - Next, mark each disk as physical volumes that can be used to create logical volumes: 
 ```
 sudo pvcreate /dev/xvdf
 sudo pvcreate /dev/xvdg
 sudo pvcreate /dev/xvdh
 ```
 To view physical volumes:
 ```
 sudo pvs
 ```
 
 <img width="423" alt="pv_create_successful" src="https://user-images.githubusercontent.com/23315232/123275684-7255ba00-d4fc-11eb-9340-09d3116a4b8f.png">
 
 - Next, add the three physical volumes to a volume group named 'webdata-vg' to be used to create logical volumes
 ```
 sudo vgcreate webdata-vg /dev/xvdf /dev/xvdg /dev/xvdh
 ```
 - Use `lvcreate` utility to create two logical volumes named apss-lv and logs-lv respectively each half the size of the volume group size
 ```
 sudo lvcreate -n apps-lv -L 14G webdata-vg
 sudo lvcreate -n logs-lv -L 14G webdata-vg
 ```
 
 <img width="592" alt="lvcreated_succesfully" src="https://user-images.githubusercontent.com/23315232/123280986-307b4280-d501-11eb-8e57-e6a50786b64d.png">
 
 - Next format the logical volumes with ext4 file system
```
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
- Create /var/www/html directory to store web files and /home/recovery/logs directory to backup log data
```
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
```
- Mount apps-lv logical volume at /var/www/html
```sudo mount /dev/webdata-vg/apps-lv /var/www/html
```
- Backup the contents of the /var/log directory in /home/recovery/logs directory
```
sudo rsync -av /var/log/. /home/recovery/logs/
```
- Mount the logs-lv volume at /var/log
```
sudo mount /dev/webdata-vg/logs-lv /var/log
```

- Restore the logs back into /var/log
```
sudo rsync -av /home/recovery/logs/ /var/log
```
- To persist the mount configuration, add the UUID of the devices to the /etc/fstab file. The UUID (Universal Unique Identifiers) for the devices can 
be gotten using the command `blkid` 

<img width="589" alt="adding_mount_point_to_fstab_config_file" src="https://user-images.githubusercontent.com/23315232/123534110-8bd04f00-d712-11eb-8ec4-e7a600759f91.png">

- Test the configuration and reload the daemon
```
sudo mount -a
sudo systemctl daemon-reload
```

The mounted devices are shown below using the command `df -h`:

<img width="499" alt="mount_point_setup_complete" src="https://user-images.githubusercontent.com/23315232/123534192-30eb2780-d713-11eb-953f-f287e44bcc66.png">
 
## Step 2 - Prepare the Database Server
Launch a second RedHat Ec2 instance that will serve as the DB-Server and perform the same steps as was done on the web-server. The logial volumes are named db-lv and log-lv and are mounted at /db and /var/log directories respectively.

<img width="511" alt="dbserver_lvs_mounting_verification" src="https://user-images.githubusercontent.com/23315232/123534322-4a40a380-d714-11eb-9e04-2f1cde209391.png">

## Step 3 - Install Wordpress on the web server
- Update the repository
```
sudo yum -y update
```
- Install wget Apache and its dependencies
```
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
- Start Apache service
```
sudo systemctl enable httpd
sudo systemctl start httpd
```
- To install php and its dependencies
```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1
```

- 
 
 


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

 
 
 
 
 


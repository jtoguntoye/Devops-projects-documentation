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
be gotten using the command `blkid`. The /etc/fstab file is shown below:

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

- Restart Apache 
```
sudo systemctl restarst httpd
```
- Next, download wordpress and copy it to /var/www/html directory
```
mkdir wordpress
cd   wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
cp -R wordpress /var/www/html/
```
- Configure SELinux (Security Enhanced Linux) policies
```
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
 sudo setsebool -P httpd_can_network_connect=1
```
 
 ## Step 4- Install MySQL on the DB server EC2
 ```
 sudo yum update
 sudo yum install mysql-server
```
- Start the mysql service
```
sudo systemctl restart mysqld
sudo systemctk enable mysqld
```
The Status of the mysql service is shown below:

<img width="679" alt="db_server_mysqld_status" src="https://user-images.githubusercontent.com/23315232/123552765-8523f500-d76f-11eb-88a9-e66ce336924e.png">

## Step 5 - Configure DB to work with Wordpress
- create 'wordpress' database in mysql, create a new user identified as connecting from the webserver's private IP address with a password, grant the create user all privileges on the wordpress db
```
sudo mysql
CREATE DATABASE wordpress;
CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```
The created db is shown below:

<img width="609" alt="db_server_lv_creation" src="https://user-images.githubusercontent.com/23315232/123552932-84d82980-d770-11eb-823a-1c71d39ca04e.png">

## Step 6 Configure Wordpress to connect with remote database
On AWS, open MYSQL port 3306 on DB server EC2 instance

<img width="788" alt="db-server-inbounr_rule" src="https://user-images.githubusercontent.com/23315232/123553138-67578f80-d771-11eb-971f-f03a69d21cdf.png">
- Install mysql-client on the web server, and update SELinux config to allow the web server to use a remote database
```
sudo yum install mysql
sudo setsebool -P httpd_can_network_connect_db on
sudo mysql -u myuser -p -h <DB-server-private-IP-address>
```
- Update the wp-config.php file  to allow the web server connect to the remote db

<img width="447" alt="wp-config php file" src="https://user-images.githubusercontent.com/23315232/123553647-58bea780-d774-11eb-997f-b6a4a822c324.png">

The databases shown by connecting to the remote db server:

<img width="507" alt="remote_login_into_dbserver_from_web_server" src="https://user-images.githubusercontent.com/23315232/123553695-bc48d500-d774-11eb-8117-ba7361033ea8.png">

- Open TCP port 80 on the web server

-Access wordpress from a browser using the public address of the web server
`http://<Web-Server-Public-IP-Address>/wordpress/`

<img width="945" alt="wordpress_installed" src="https://user-images.githubusercontent.com/23315232/123553782-282b3d80-d775-11eb-946d-4fab6b0a3257.png">

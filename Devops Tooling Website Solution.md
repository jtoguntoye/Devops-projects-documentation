# DevOps Tooling Website Solution
- Aim: Implementing a tooling website solution which displays a dashbord of useful DevOps tools.

## Set up and Technology used
- Infrastructure: AWS
- Web server: Red Hat Linux 8
- Database Server: Ubuntu 20.04 + MySQL
- Storage Server: Red Hat Enterprise Linux 8 + NFS Server
- Programming Language: PHP


### Prepare NFS server
#### Spin up a new EC2 instance on AWS with RHEL 8 OS
#### Configure logical volumes on the server
  - Add EBS (Elastic Block Storage) volumes to the Ec2 instance to increase storage space
  - Attached Block devices are shown below:
  
  <img width="403" alt="block_Devices_on_nfs_Server_lsblk" src="https://user-images.githubusercontent.com/23315232/124399889-79f43a80-dd16-11eb-8c99-0805729f7f8a.png">
  
  - Create three logical volumes on the NFS server of 'XFS' format. The LVS are named - `lv-apps`, `lv-logs` and `lv-opt`
  - Create mount points for the logical volumes in /mnt directory: lv-apps on `/mnt/apps` for web server, lv-logs on `/mnt/logs` for webserver logs, and lv-opt on  `/mnt/opt` to be used by Jenkins server in project 8

<img width="485" alt="nfs_server_devices_showing_logical_vols" src="https://user-images.githubusercontent.com/23315232/124400370-cc832600-dd19-11eb-921f-4d5da27fded9.png">

<img width="606" alt="nfs_Server_formatting_lvs_as_xfs" src="https://user-images.githubusercontent.com/23315232/124400134-53cf9a00-dd18-11eb-80a9-a72e015258ef.png">

<img width="525" alt="nfs_server_verifying_lv_mount_points" src="https://user-images.githubusercontent.com/23315232/124400139-59c57b00-dd18-11eb-951b-d6c5db1bcb19.png">

 - Install NFS server, configure it to restart on reboot and confirm to make sure it is up and running
 ```
 sudo yum -y update
 sudo yum instsall nfs-utils-y
 udo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
 ```
 
<img width="770" alt="nfs_Server_starting_nfs_service" src="https://user-images.githubusercontent.com/23315232/124400799-29341000-dd1d-11eb-828f-57949fc48584.png">

- Export the logical volume mounts of the NFS server  for web servers' subnet CIDR to connect as clients. (In production, you would want to setup the each tier in your infrastructure with different subnet CIDR for higher level of security) 
 To export the mounts of the NFS server, add the mounts to the `/etc/exports` file with the desired options for the exported mounts. Then update the list of exported directories with the command  `sudo exportfs -arv`
 
 <img width="399" alt="nfs_Server_exports_file" src="https://user-images.githubusercontent.com/23315232/124401034-c479b500-dd1e-11eb-8a24-2310176fd431.png">
 
 
- Next, set up permissions that will allow the webs servers to read and write to the NFS serverand restart NFS service.
```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```
- Check which ports are being used by NFS server using the command ``` sudo rpcinfo -p | grep nfs```. Also in order for NFS server to be accessible from the client, open the following ports on the NFS server: TCP 111, UDP 111, UDP 2049

<img width="772" alt="nfs_server_inbound_rules_update" src="https://user-images.githubusercontent.com/23315232/124401359-f68c1680-dd20-11eb-8132-0ef8c319add9.png">

### Configure the Database server
- Spin up an EC2 instance of type Ubuntu 20.04 for the database server
- Install MySQL server
```
sudo apt install mysql-server -y
```
```
sudo apt install mysql_secure_installation
```
- Log in to MYSQL and create a database named `tooling`
- Next create a new user named `webaccess` and grant permissions to the user on the tooling db only from the web server's subnet CIDR
```
sudo mysql
CREATE DATABASE tooling;
CREATE USER `webaccess`@`<Web-Server-SUBNET-CIDR>` IDENTIFIED BY 'password';
GRANT ALL ON wordpress.* TO 'webaccess'@'<Web-Server-SUBNET-CIDR>';
FLUSH PRIVILEGES;
```

<img width="219" alt="db_server_user_table" src="https://user-images.githubusercontent.com/23315232/124463984-89619b00-dd8b-11eb-81b5-92f5b74923ff.png">

### Configure web servers
-Spin up 3 EC2 instances of type Red Hat Linux 8 for the web servers. The 3 web servers will have the same subnet CIDR.
- Install NFS client:
```
sudo yum install nfs-utils nfs4-acl-tools -y
```
- Create `var/www/` directory and mount the lv-apps logical volume exported from the NFS server on the directory
```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
```
- Persist the changes to the web server to make the mounts available after reboot by adding the NFS lv to the /etc/fstab file
``` <NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0 ```

- The mount on the web server is shown using `df -h` below:

<img width="399" alt="mounting_exported_folder_on_web_server" src="https://user-images.githubusercontent.com/23315232/124468813-7eaa0480-dd91-11eb-9cb1-27d8a8b28834.png">

- Install Apache on the web servers
```
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
``` 

- Once installed on the web server the Apache files and directories also become available on the NFS server.

<img width="295" alt="sync_between_web_Servers_and_nfs_Server" src="https://user-images.githubusercontent.com/23315232/124483107-e0726a80-dda1-11eb-9f67-c91c0692cba3.png">
- Next mount the NFS export for logs `lv-logs` on the `/var/logs` directory of the web server.

<img width="446" alt="mount_nfs_exported_log_on_web_Server's_httpd_log_directory" src="https://user-images.githubusercontent.com/23315232/124494878-474a5080-ddaf-11eb-947e-36e3ba91b92a.png">

- Install git and clone the tooling source code at https://github.com/darey-io/tooling.git
```
sudo yum install git -y
git clone https://github.com/darey-io/tooling.git
```
- Next, the tooling repository code is deployed to /var/www/html folder

<img width="858" alt="copying_git_clone_html_folder_to_var_html1" src="https://user-images.githubusercontent.com/23315232/124495348-eb33fc00-ddaf-11eb-8a99-0f9fd596b823.png">

<img width="476" alt="accessing_web_server" src="https://user-images.githubusercontent.com/23315232/124496855-e5d7b100-ddb1-11eb-9702-45ded503302e.png">



- Install php on the web server.
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
- Restart httpd service
```
sudo systemctl restart httpd
```
- Update the functions.php file to be able to connect to the remote database (by updating the IP address, username, password and database name), and apply the tooling-db.sql script to the database

<img width="634" alt="functions php file" src="https://user-images.githubusercontent.com/23315232/124500233-4a493f00-ddb7-11eb-9c00-31b88185cd76.png">

```
sudo mysql -u myaccess -h <private-ip-address-of-db-server> -p < tooling-db.sql
```

- On the DB server, add a new user named  to the user table of the tooling database
```
INSERT INTO `users` (
    `id`,
    `username`,
    `password`,
    `email`,
    `user_type`,
    `status`
  )
VALUES (
    2,
    'myuser',
    'PASS',
    'user@mail.com',
    'admin',
    '1'
  );
```
- On the web server the tooling website login page is accesible at the web server's public IP address 

<img width="697" alt="login_php" src="https://user-images.githubusercontent.com/23315232/124501376-533b1000-ddb9-11eb-987c-4e51f8f5f956.png">

- Log in with the created username and password
<img width="925" alt="propitix_home_tooling_website" src="https://user-images.githubusercontent.com/23315232/124501424-6d74ee00-ddb9-11eb-8d7a-76769710fc71.png">

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

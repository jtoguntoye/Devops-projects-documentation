# Load Balancer Solution with Apache

## Task- Deploy and configure an Apache load balancer for Tooling website (deployed in project 7) on a separate Ubuntu EC2 instance

## Architecture

<img width="337" alt="architecture_for_project" src="https://user-images.githubusercontent.com/23315232/125077318-de1d5280-e0b8-11eb-9a5c-bde561491b70.png">
          Source: Darey.io



## Prerequisite configuration
- From setup in project 7:
Two RHEL8 Web Servers
One MySQL DB Server (based on Ubuntu 20.04)
One RHEL8 NFS server

- Apache up and running on both web servers, NFS server configured, /var/www directory of both web servers mounted to /mnt/apps of NFS server, 
all necessary UDP/TCP ports are opened on the web, DB and NFS servers.

## Configure Apache as a load balancer
- Create an Ubuntu server 20.04 EC2 instance named `project-8-apache-lb`.
- Running Ec2 list is shown below:

<img width="788" alt="EC2_instances_running" src="https://user-images.githubusercontent.com/23315232/124840233-6d741a00-df82-11eb-8803-6076681d877b.png">

- Open TCP port 80 on the server:
<img width="775" alt="security_group_open_port_80" src="https://user-images.githubusercontent.com/23315232/124840613-4d912600-df83-11eb-9ca2-a92d4d4bf602.png">

- Install Apache load balancer on `project-8-apache-lb`
```
#install apache
sudo apt update 
sudo apt install apache2 -y
sudo apt install libxml2-dev

#enable the following modules
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy-balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic

#Restart apache2 service
sudo systemctl restart apache2
```
- Configure load balancing
  Add the private IP addresses of the two webservers to the config file for  `/etc/apache2/sites-available/000-defaulf.conf`
  
  ``` sudo vi /etc/apache2/sites-available/000-defaulf.conf```
  
  <img width="696" alt="add_web_servers_to_load_balancer_config" src="https://user-images.githubusercontent.com/23315232/124841107-69e19280-df84-11eb-83fa-8665196a5002.png">

- Restart Apache server
 ```
 sudo systemctl restart apache2
 ```
 
 <img width="575" alt="apache2_service_Status" src="https://user-images.githubusercontent.com/23315232/124841871-2d169b00-df86-11eb-87af-110083174570.png">
 
 - Verify that the configuration works by accessing the load balancer's public DNS address in the browser
 
 <img width="919" alt="propitix_index_page" src="https://user-images.githubusercontent.com/23315232/124841823-11ab9000-df86-11eb-9c2c-46a8d606f04d.png">
 
 - Viewing access logs on the two web servers after refreshing the browser page for the LB several times:
 
 ```
 sudo tail -f /var/log/httpd/access_logs
 ```
 
 <img width="911" alt="tail-f_httpd_access_log_web_Server1" src="https://user-images.githubusercontent.com/23315232/124842547-b2e71600-df87-11eb-8eaf-d48757319c3d.png"> 
 
<img width="691" alt="tail-f_httpd_access_log_web_Server2" src="https://user-images.githubusercontent.com/23315232/124842391-5683f680-df87-11eb-8125-bc3bab583039.png">

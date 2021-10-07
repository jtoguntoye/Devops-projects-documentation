# AWS Solution for a 2-company Website Using Reverse Proxy Technology


## Objective:
- Buuild a secure infrastructure in AWS VPC for a company that uses Wordpress CMS for its main company site and a tooling website for its DevOps team. 
- Reverse proxy technology with Nginx has been selected with the aim to improve security and performance.
- Cost, reliability and scalability are the major considerations for the company, to make the infrastructure resilient to server failures, accomodate increased traffic   and have reasonable cost.

Infrastructure:

![project-15-infrastructure IMG](https://user-images.githubusercontent.com/23315232/135461812-e94e31b1-526b-4950-b82a-e910ee53773c.png)
Credits: Darey.io

### Initial setup
- Create a subaccount in AWS to manage all resources for the company's AWS solution and assign any appropriate name e.g 'DevOps'
- From the root account, create an organization unit (OU) and move the subaccount into the OU. We will launch Dev resources in the subaccount

<img width="938" alt="creating_organization_unit_ou_and_Adding_devops_account" src="https://user-images.githubusercontent.com/23315232/135610014-f847f9ce-e04e-4ba6-be4e-a84f0b77c2c5.png">

- Create a domain name for the company website on domain name providers. You can obtain a free domain name from freenom website
- Create a hosted zone in AWS and map the hosted zone name servers to the domain name.

<img width="639" alt="creating_route_53_hosted_zone_in_aws" src="https://user-images.githubusercontent.com/23315232/135609723-01b7ba48-f7ac-4f63-9d34-dc98716a8938.png">

<img width="663" alt="mapping_hosted_zone_name_Servers_to_domain_name" src="https://user-images.githubusercontent.com/23315232/135609786-8664ade8-a286-4e42-b4a4-ec24d701dbeb.png">

### Setup a Virtual Private Cloud on AWS
- Create a VPC 
- Create subnets (public and private subnets) as shown in the architecture. The subnets Ips are CIDR IPs. We can use utility sites like IPinfo.io to see the range of IP addresses associated with each subnet.

<img width="772" alt="Public-subnet-in-two-AZs" src="https://user-images.githubusercontent.com/23315232/135610761-69a92e4b-af61-47f8-90fd-bc47c1928083.png">

- Create private and public route tables and associate it with with the private and public subnets respectively

<img width="938" alt="edit_route_in_public_routetable_to_allow_subnets_access_the_internet" src="https://user-images.githubusercontent.com/23315232/135612563-2b40aed3-5308-4d0a-94bc-a4d8f1dfa33d.png">

- Edit a route in public route table, and associate it with the Internet Gateway. This allows the public subnet to access the internet

<img width="938" alt="edit_route_in_public_routetable_to_allow_subnets_access_the_internet" src="https://user-images.githubusercontent.com/23315232/135612563-2b40aed3-5308-4d0a-94bc-a4d8f1dfa33d.png">

- Create a NAT gateway and assign an elastic IP to it. You can use a NAT gateway so that instances in a private subnet can connect to services outside your VPC but external services cannot initiate  a connection with those instances.

<img width="937" alt="edit_route_in_private_routetable_to_allow_nat_Gateway_access_the_internet" src="https://user-images.githubusercontent.com/23315232/135612545-4c9cf9c3-deb2-4d4e-9fb9-cb38feb6c9d6.png">

- Create security groups for:
  -  Nginx servers: To allow access to from external application load balancer to the Nginx server
  -  Bastion servers: Access to the Bastion servers should be allowed only from workstations that need to SSH into the bastion servers. 
  -  External Load Balancer: The external application load balancer will be accessible from the internet
  -  Internal load balancer: The internal load balancer will allow https and http access from the  Nginx server and SSH access from the bastion server
  -  Web servers: The webservers will allow https and http access from the internal load balancer and SSH access from the bastion server
  -  Data layer security group: The access to the data layer for the appliation (consisting of both the Amazon RDS storage and Amazon EFS as shown in the architecture), will consist of webserver access to the RDS storage and both webserver and Nginx access to the Amazon EFS file system.
   
  <img width="900" alt="security_grp_rule_for_bastion_host" src="https://user-images.githubusercontent.com/23315232/135623670-bbbd20e3-944e-45e9-9054-05beef0349d2.png">
  
  <img width="904" alt="security_grp_rule_for_External_ALB" src="https://user-images.githubusercontent.com/23315232/135623681-1b0f9961-f807-4d6b-85d5-9b7a069ed7ff.png">

### Create a SSL/TLS certificate using Amazon Certificate Manager (ACM) to be used by the external and internal Application Load balancers (ALB)
- Create a wild card SSL/TLS certificate to be used when creating the external ALB. We want to ensure connection to the external ALB is secured and data sent over the internet is encrypted. Since the external ALB will listen for client requests to both the tooling webserver and the wordpress server, we'll create a wild card TLS certificate. Select DNS validation 

<img width="927" alt="creating_public_wild-card-TLS_certificate_for_ALBs" src="https://user-images.githubusercontent.com/23315232/135927685-23d76672-0d20-4f98-8f00-2466294515cd.png">

### Create Amazon EFS
- Create Amazon Elastic file System (EFS) to be used by the web servers for files storage. The mount targets to be specified fot the elastic file system will be the subnets for the webservers. Specifying the mount targets makes the EFS storage available to the webservers

<img width="729" alt="creating-Amazon-EFS-for-the-wordpress-and-tooling-servers-to-access" src="https://user-images.githubusercontent.com/23315232/136053493-6a69cb91-1a77-42d1-8af7-fbbaa7d4e7bb.png">

- Also, we specify access points on the EFS we created for the web servers. Amazon EFS access points are application-specific entry points into a shared file system.     In this project, we create two access points on the EFS one for each web servers, each with its own root directory path specified.  Set the POSIX user and user group   ID to root user and the root directory path to ```/wordpress``` and `/tooling` respectively.
- The root directory creation permission is set to `0755` to allow read write permissions on the file system by the clients 


<img width="726" alt="tooling-access-point-created on-EFS-for-the-tooling-webserver" src="https://user-images.githubusercontent.com/23315232/136055726-9aea4cd5-2fe9-4001-aa93-39d2ec6cf6ea.png">

### Create KMS key to be used for RDS
- Next, navigate to AWS KMS page to create a cryptographic key to be used to secure the MySQL relational database for the project.
- Create a symmetric key
- Set the admin user for the key. You can leave the 'key usage permission' with the default settings

<img width="663" alt="creating-symmetric-key-for-encrypting-and-decrypting-the-DB" src="https://user-images.githubusercontent.com/23315232/136060813-6fcb5b53-6283-49ce-b20f-e7815bbfdac2.png">


<img width="916" alt="kms-key" src="https://user-images.githubusercontent.com/23315232/136060837-c38d6b07-5311-4ff2-aac0-e0e2677f0403.png">

### Create DB subnet group
- A DB subnet group is a collection of subnets (typically private) that you create for a VPC and that you then designate for your DB instances.
- From project architecture, specify the appropriate private subnets (private subnet 3 and 4) for the DB.  

<img width="926" alt="creating_subnet_group_for_RDS" src="https://user-images.githubusercontent.com/23315232/136171766-9b721acb-e079-4c9b-be83-027927f1d935.png">

### Create AWS RDS
- Select MySQL engine for the RDS
- Select Dev/Test template. This is an expensive service. For this purpose of this project we can still use free tier template, however we will not be able to encrypt the database using the KMS key we created.
-  set the DB name
-  Set master username and password
-  Select the VPC for the DB
-  Ensure the DB is not publicly accessible
-  Select the appropriate security group for the DB
-  set the inital database name 

<img width="896" alt="creating_DB" src="https://user-images.githubusercontent.com/23315232/136179589-e7f82a18-39f6-4e42-ba60-ac0b64eeb74a.png">

### Create compute resources

  #### Setup compute resources for Nginx
  - provision EC2 instance for Nginx
  - Install the following packages
    ```
    epel-release
    python
    htop
    ntp
    net-tools
    vim
    wget
    telnet
    ```
  - We also need to install a self signed SSL certificate on the Nginx AMI. The Nginx AMI will be attached to a target group that uses HTTPs protocol and health           checks. The load balancer establishes TLS connections with the targets using certificates that you install on the targets 
  - Nginx instance installations:

  ```
  yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
  yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
  yum install wget vim python3 telnet htop git mysql net-tools chrony -y
  systemctl start chronyd
  systemctl enable chronyd
  
  #configure SELinux policies
  setsebool -P httpd_can_network_connect=1
  setsebool -P httpd_can_network_connect_db=1
  setsebool -P httpd_execmem=1
  setsebool -P httpd_use_nfs 1
  
  #install Amazon efs client utils
  git clone https://github.com/aws/efs-utils
  cd efs-utils
  yum install -y make
  yum install -y rpm-build
  make rpm 
  yum install -y  ./build/amazon-efs-utils*rpm
  
  #setup self-signed certificate for the Nginx AMI
  sudo mkdir /etc/ssl/private
  sudo chmod 700 /etc/ssl/private
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/kiff.key -out /etc/ssl/certs/kiff.crt
  sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
  ```
  - We will reference the SSL key and cert in the Nginx `Reverse.conf` configuration file. Also we will specify host headers in the config file to forward traffic to
    the tooling server. 
    
  Nginx reverse.conf file
  ```
  ```
  - We perform the above installations on the EC2 instance for the AMI step by step instead of adding them all to the launch template's user data, to reduce to size of 
    the user data
    
  - Create an AMI from the instance
  - Create a Nginx target group of Instance type. Targets in the Nginx target groups will be accessed by the external Load balancer. 
     
  - Prepare a launch template from the AMI instance
  - From EC2 Console, click Launch Templates from the left pane
    - Choose the Nginx AMI
    - Select the instance type (t2.micro)
    - Select the key pair
    - Select the security group
    - Add resource tags
    - Click Advanced details, scroll down to the end and configure the user data script to update the yum repo and install nginx. The userdata for Nginx:
   
   ```
   #!/bin/bash
   yum install -y nginx
   systemctl start nginx
   systemctl enable nginx
   git clone https://github.com/joeloguntoyeACS-project-config.git
   mv /ACS-project-config/reverse.conf /etc/nginx/
   mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf-distro
   cd /etc/nginx/
   touch nginx.conf
   sed -n 'w nginx.conf' reverse.conf
   systemctl restart nginx
   rm -rf reverse.conf
   rm -rf /ACS-project-config
   ```

#### Setup compute resources for Bastion server
- Provision EC2 instance for Bastion server
- Bastion instance installations:

```
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm 
yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm 
yum install wget vim python3 telnet htop git mysql net-tools chrony -y
systemctl start chronyd
systemctl enable chronyd
```

### Connect to the RDS from the Bastion server and create DBs named toolingdb and wordpressdb for the two webservers
- SSH into the RDS instance 
```
eval `ssh-agent`
ssh-add project-key.pem
ssh -A ec2-user@ip_address
mysql -h <RDS_endpoint> -u <username> -p 
>>create database wordpressdb;
>>create database toolingdb;
```


<img width="256" alt="creating_databases_on_RDS_using_bastion_server_to_access_RDS" src="https://user-images.githubusercontent.com/23315232/136358127-61ad4208-6064-43a4-baf8-0ba570925577.png">

### Setup compute resources for web server
- Provision EC2 instance for web servers
- Web server installations:
```
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
yum install wget vim python3 telnet htop git mysql net-tools chrony -y
systemctl start chronyd
systemctl enable chronyd
 
#configure SELinux policies
setsebool -P httpd_can_network_connect=1
setsebool -P httpd_can_network_connect_db=1
setsebool -P httpd_execmem=1
setsebool -P httpd_use_nfs 1

#install Amaxzon EFS client utils for mounting the targets on the EFS
git clone https://github.com/aws/efs-utils
cd efs-utils
yum install -y make
yum install -y rpm-build
make rpm 
yum install -y  ./build/amazon-efs-utils*rpm

#setup self-signed certificate for apache server
yum install -y mod_ssl
openssl req -newkey rsa:2048 -nodes -keyout /etc/pki/tls/private/kiff-web.key -x509 -days 365 -out /etc/pki/tls/certs/kiff-web.crt

# edit the ssl.conf file to specify the part to the certificate and the key
vi /etc/httpd/conf.d/ssl.conf
```
- Create an AMI from the instance 
- We will create two launch templates from this AMI, one each for the wordpress server and the tooling server. The launch templates will differ in the user data for     each server. The launch tmeplate 
-  Configure user data for the worpress launch template:
```
#!/bin/bash
mkdir /var/www/
sudo mount -t efs -o tls,accesspoint=fsap-0f9364679383ffbc0 fs-8b501d3f:/ /var/www/
yum install -y httpd 
systemctl start httpd
systemctl enable httpd
yum module reset php -y
yum module enable php:remi-7.4 -y
yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
systemctl start php-fpm
systemctl enable php-fpm
wget http://wordpress.org/latest.tar.gz
tar xzvf latest.tar.gz
rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
mkdir /var/www/html/
cp -R /wordpress/* /var/www/html/
cd /var/www/html/
touch healthstatus
sed -i "s/localhost/kiff-database.cdqtynjthv7.eu-west-3.rds.amazonaws.com/g" wp-config.php 
sed -i "s/username_here/Kiffadmin/g" wp-config.php 
sed -i "s/password_here/admin12345/g" wp-config.php 
sed -i "s/database_name_here/wordpressdb/g" wp-config.php 
chcon -t httpd_sys_rw_content_t /var/www/html/ -R
systemctl restart httpd
```
- Configure user data for tooling launch template:

```
#!/bin/bash
mkdir /var/www/
sudo mount -t efs -o tls,accesspoint=fsap-01c13a4019ca59dbe fs-8b501d3f:/ /var/www/
yum install -y httpd 
systemctl start httpd
systemctl enable httpd
yum module reset php -y
yum module enable php:remi-7.4 -y
yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
systemctl start php-fpm
systemctl enable php-fpm
git clone https://github.com/Livingstone95/tooling-1.git
mkdir /var/www/html
cp -R /tooling-1/html/*  /var/www/html/
cd /tooling-1
mysql -h kiff-db.cdqpbjkethv0.us-east-1.rds.amazonaws.com -u kiffAdmin -p toolingdb < tooling-db.sql
cd /var/www/html/
touch healthstatus
sed -i "s/$db = mysqli_connect('mysql.tooling.svc.cluster.local', 'admin', 'admin', 'tooling');/$db = mysqli_connect('kiff-db.cdqpbjkethv0.us-east-1.rds.amazonaws.com ', 'kiffAdmin', 'admin12345', 'toolingdb');/g" functions.php
chcon -t httpd_sys_rw_content_t /var/www/html/ -R
systemctl restart httpd
```
<img width="555" alt="configuring_user_Data_for_bastion_launch_template" src="https://user-images.githubusercontent.com/23315232/136201972-5e10fd0d-5990-4f4e-b0c8-4d984d443f54.png">

Target group IMG:

<img width="940" alt="creating_target_groups_for_nginx_tooling_and_wordpress_servers_to_be_Targeted_by_ALBs" src="https://user-images.githubusercontent.com/23315232/136201615-859a95a3-d5b5-47ab-8e57-2cacaeff62f5.png">

### Create load balancers (the external load balancer and the internal load balancer)

- Create external load balancer. 
  - Assign  at least two public subnets
  - set protocol as HTTPs on port 443
  - Register the Nginx target group for the external load balancer
  - select the security group for the external load balancer
  - set path for healthchecks as `/healthstatus`
  
- Create the internal load balancer
  - Assign at least two private subnets
  - set protocol as HTTPs on port 443
  - select the security group for the internal load balancer
  -  set path for healthchecks as `/healthstatus`
  - Register the wordpress target group as the default target for the internal load balancer
  - Configure a listener rule to allow the internal load balancer forward  traffic to the tooling target group based on the rule set.
  - Since we'll configure host header in our Nginx reverse proxy server, we will specify the listener rule on the ALB to forward traffic to the tooling target if the       host header is the domain name : `tooling.kiff-web.space` 
  
  <img width="958" alt="creating_ext_and_int_load_balancers_with_listener_rule_set_for_internal_lb" src="https://user-images.githubusercontent.com/23315232/136204901-af0c598d-3c5b-4c36-8a42-b2c50df52cad.png">
  
  
  <img width="863" alt="configuring_listener_rule_for_internal_LB" src="https://user-images.githubusercontent.com/23315232/136206252-03c29298-7b2d-44a6-92a0-546cd6e7bc35.png">

### Create Autoscaling groups for the launch templates (Bastion, Nginx, tooling and wordpress servers)
- Configure Autoscaling for Nginx
  - Select the right launch template
  - Select the VPC
  - Select both public subnets
  - Enable Application Load Balancer for the AutoScalingGroup (ASG)
  - Select the Nginx target you created before
  - Ensure that you have health checks for both EC2 and ALB
  - The desired capacity is 2
  - Minimum capacity is 2
  - Maximum capacity is 4
  - Set scale out if CPU utilization reaches 90%
  - Ensure there is an SNS topic to send scaling notifications 
  
- Configure Autoscaling For Bastion
  - Select the right launch template
  - Select the VPC
  - Select both public subnets
  - Select No load balancer for bastion Autoscaling group, since the Bastion server is not targeted by any load balancer  
  - Set scale out if CPU utilization reaches 90%
  - Enable health checks
  - The desired capacity is 2
  - Set Minimum capacity to 2
  - Maximum capacity to 4
  - Ensure there is an SNS topic to send scaling notifications
  
  <img width="895" alt="creating_bastion_AG_review" src="https://user-images.githubusercontent.com/23315232/136360585-a1791039-81ef-498d-8b70-c49d853fad72.png">
 
 - Configure Autoscaling group for tooling and wordpress webserver 
  - Select the right launch template
  - Select the VPC
  - Select both private subnets  1 and 2
  - Enable Application Load Balancer for the AutoScalingGroup (ASG)
  - Select the target groups you created before
  - Ensure that you have health checks for both EC2 and ALB
  - The desired capacity is 2
  - Minimum capacity is 2
  - Maximum capacity is 4
  - Set scale out if CPU utilization reaches 90%
  - Ensure there is an SNS topic to send scaling notifications
  
  
### Create A Records in the Route 53 hosted zone
- create A record for tooling and wordpress 
  - set record type to 'A-Routes to a IPV4 address' 
  - Set Route Traffic to 'Alias to application load balancer and classic load balancer'
  - Set the load balancer
  
  <img width="707" alt="adding_A_record_to_DNS" src="https://user-images.githubusercontent.com/23315232/136365772-dda6ce29-d45f-4d26-8c18-7f9edbdc1a5d.png">
  
  
  
  Healthchecks status for wordpress targets:
 
  <img width="703" alt="showing_health_checks_for_ALB_targets" src="https://user-images.githubusercontent.com/23315232/136360518-f891e40b-c4d4-4f52-9ef1-566faf9a745a.png">

  Accessing the tooling and wordpress servers:
  
  <img width="923" alt="wordpress_server_page_loaded" src="https://user-images.githubusercontent.com/23315232/136368278-574940c3-dd33-4a36-ac25-4ebd26356ab0.png">
  


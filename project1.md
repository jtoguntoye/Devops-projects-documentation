## WEB STACK IMPLEMENTATION (LAMP STACK) IN AWS

### Step 1
First, I Set up AWS account and launched an EC2 instance with Ubuntu OS on it. I downloaded the private key file (.PEM file) for the EC2 instance. This file 
will be needed to establish a remote connection (SSH) into the EC2 instance. 

### Step 2
I Connected to the EC2 instance using MobaXterm, created a new session using the Public DNS name and the private key for the EC2 instance   
<img width="609" alt="mobaxterm" src="https://user-images.githubusercontent.com/23315232/119142522-852d2880-ba3e-11eb-8dd5-02c45444568a.png">

To resolve connection timed out error: I edited the inbound rules on for the EC2 instance to allow SSH connection from all IPs following 
the instructions in [this link](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html).

Below is a screenshot of MobaXterm ui after establishing connection:

<img width="478" alt="ssh_img" src="https://user-images.githubusercontent.com/23315232/119143398-75621400-ba3f-11eb-8c2e-6eec940ef585.png">

### Step 3
Next, I updated apt package manager and installed Apache2 HTTP server on the EC2 instance from the remote terminal with the commands: 
```
sudo apt update 
sudo apt install apache2 
```
Screenshot of Apache status check:

<img width="472" alt="apachepic" src="https://user-images.githubusercontent.com/23315232/119146193-4f8a3e80-ba42-11eb-87ab-ac06aead950e.png">

Next, to allow web traffic on the server, I added a TCP inbound rule to the EC2 instance on AWS console.

### Step 4 
I installed MySQL on the EC2 instance with the command:
```
sudo apt apt install mysql-server
```
I removed insecure default settings from MySQl with the command:
```
sudo mysql_secure_installation
```
Screen shot after logging in to MySQL console:

<img width="478" alt="mysql_install" src="https://user-images.githubusercontent.com/23315232/119148269-3c786e00-ba44-11eb-97a9-efa6e1fe01c0.png">

### Step 5
Next, I installed PHP with the command:
```
sudo apt install php libapache2-mod-php php-mysql
```
Screen shot of PHP version installed:

<img width="404" alt="php_install" src="https://user-images.githubusercontent.com/23315232/119148890-dcce9280-ba44-11eb-87dd-f545bb84bd79.png">


### Step 6
To Set up virtual host for website using Apache, I created a new directory with name projectlamp and 
assigned the ownership of the directory with commands:
```
sudo mkdir /var/www/projectlamp
sudo chown -R $USER:$USER /var/www/projectlamp
```
A new configuration file was created in Apache's sites-available directory with the virtual host configuration added:
```
sudo vi /etc/apache2/sites-available/projectlamp.conf
```

### Step 7
To enable PHP, i changed the order of listing of index.php file:
```
sudo vim /etc/apache2/mods-enabled/dir.conf
```

Result of connecting to ip of EC2 instance:
<img width="806" alt="php_info" src="https://user-images.githubusercontent.com/23315232/119152661-6cc20b80-ba48-11eb-9acc-82f1bc6dd39e.png">


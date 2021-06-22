# Client Server Architecture with MySQL

## Step 1
Launch two AWS EC2 instances named `mysql-server` and `mysql-client` respectively.

<img width="801" alt="aws ec2 instances" src="https://user-images.githubusercontent.com/23315232/122800368-bfe2e480-d2ba-11eb-83cd-27782110e805.png">

## Step 2
 Connect to the EC2 instance for the mysql server using mobaXterm. Next, perform package update and install mysql-server with the commands:
 ```
 sudo apt update
 sudo apt install mysql-server
 ```
 Screenshots of the successful apt update and mysql server installation are shown below:
 
 <img width="667" alt="apt-update" src="https://user-images.githubusercontent.com/23315232/122800966-7777f680-d2bb-11eb-8870-ee95ed227040.png">
 
 <img width="687" alt="mysql-server-install" src="https://user-images.githubusercontent.com/23315232/122800980-7cd54100-d2bb-11eb-8178-b3d08b366d33.png">
 
 Next, confirm the mysql-server service running with the command `systemctl status mysql`.
 
 <img width="655" alt="mysql-server-status" src="https://user-images.githubusercontent.com/23315232/122801242-c6be2700-d2bb-11eb-975c-bf863dd1ed7a.png">
 
 Next, connect to the Ec2 instance to be used as mysql client and install mysql client with the commands: `sudo apt install mysql-client`
 
 Create a new entry in The Inbound rules for the mysql-server security group allowing access to port 3306 for only the mysql-client (private IP).
 
 <img width="917" alt="mysql-server-inbound -rule" src="https://user-images.githubusercontent.com/23315232/122803873-18b47c00-d2bf-11eb-8bf3-d710717f8c44.png">

Next, configure the mysql-server to allow conection from remote hosts  editing the config file at `/etc/mysql/mysql.conf.d/mysqld.cnf`

<img width="613" alt="bind-address" src="https://user-images.githubusercontent.com/23315232/122804663-130b6600-d2c0-11eb-89b8-20c3a5eb0d44.png">




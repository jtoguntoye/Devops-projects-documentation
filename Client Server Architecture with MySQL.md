# Client Server Architecture with MySQL

## Step 1
Launch two AWS EC2 instances named `mysql-server` and `mysql-client`

<img width="801" alt="aws ec2 instances" src="https://user-images.githubusercontent.com/23315232/122800368-bfe2e480-d2ba-11eb-83cd-27782110e805.png">

## Step 2 Setup the DB server -mysql-server
 Connect to the EC2 instance for the mysql server using mobaXterm. Next, perform package update and install mysql-server with the commands:
 ```
 sudo apt update
 sudo apt install mysql-server
 ```
 Enable mysql:
 ```
 systemctl enable mysql
 ```
 
 Next, confirm the mysql-server service running with the command 
 ```
 systemctl status mysql
 ```
 
 <img width="655" alt="mysql-server-status" src="https://user-images.githubusercontent.com/23315232/122801242-c6be2700-d2bb-11eb-975c-bf863dd1ed7a.png">
 
 - Create a remote user for the database
 ```
 CREATE USER 'james'@'%' IDENTIFIED BY 'PASSword;'
 ```
 
 - Create a new database:
 ```
 CREATE DATABASE devops_mentors_db;
 ```
 
 - Grant privileges to the user:
 ```
 GRANT ALL PRIVILEGES ON devops_mentors_db.* TO 'james'@'%' WITH GRANT OPTIONS;
 ```
 
 
 <img width="219" alt="show_database" src="https://user-images.githubusercontent.com/23315232/122946421-40641c80-d371-11eb-9aaa-58cad4a193b8.png">
 
 Next, configure the mysql-server to allow conection from remote hosts  editing the config file at 
 ```
 /etc/mysql/mysql.conf.d/mysqld.cnf
 ```

<img width="613" alt="bind-address" src="https://user-images.githubusercontent.com/23315232/122804663-130b6600-d2c0-11eb-89b8-20c3a5eb0d44.png">

Restart the MYSQL service on the server to activate the new configurations
```
sudo systemctl restart mysql
```

## Step 3 Setup the db client- mysql-client

 Connect to the Ec2 instance to be used as mysql client and install mysql client with the command: 
 ```
 sudo apt install mysql-client
 ```
 
 Create a new entry in The Inbound rules for the mysql-server security group allowing access to port 3306 for only the mysql-client (private IP).
 
 <img width="917" alt="mysql-server-inbound -rule" src="https://user-images.githubusercontent.com/23315232/122803873-18b47c00-d2bf-11eb-8bf3-d710717f8c44.png">
 
 ## Step 4 Connect to the db server from the remote client
 Connect to the db server from the remote client using the command below and enter the password for the user when prompted 
 ```
 mysql -u james -h 172.31.25.8 -p
 ```

 - To show database:
 ```SHOW DATABASE;
 ```
 
 <img width="413" alt="show_db_client" src="https://user-images.githubusercontent.com/23315232/122953057-54f6e380-d376-11eb-976b-3d02b3dcc211.png">




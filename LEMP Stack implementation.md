# LEMP Stack implementation

## Step 1
Launched an EC2 instance on AWS with Ubuntu OS on it. Next, I setup SSH connection to the EC2 instance with the private key using MobaXterm.
Also configured inbound security rules on the AWS EC2 instance to allow SSH connection on port 22 and HTTP connection on port 80.

<img width="799" alt="sec_rules" src="https://user-images.githubusercontent.com/23315232/119241566-5488e400-bb4f-11eb-9de9-3e9bedd908e6.png">

## Step 2
Updated the apt package manager on the remote server and installed Nginx server with the commands:
```
  sudo apt update
  sudo apt install nginx
  ```
  Nginx status after installation:
  
  <img width="497" alt="nginx _status" src="https://user-images.githubusercontent.com/23315232/119241672-0de7b980-bb50-11eb-8f4f-780507e87fe9.png">

 ## Step 3
 Installed MySql and ran script to remove insecure default settings with the commands:
 ```
 sudo apt install mysql-server
 sudo mysql_secure_installation
 ```
 After installation, result of connecting to mySQL server is shown below:
 
 <img width="477" alt="mysql_connection" src="https://user-images.githubusercontent.com/23315232/119241761-c150ae00-bb50-11eb-8f5b-e9bb24cd7b2f.png">
 
 ## Step 4
 Installed PHP and configured Nginx server to use PHP processor. After saving the configuration file. The result of checking the configuration
 for syntax error is shown below:
 
 <img width="497" alt="nginx _status" src="https://user-images.githubusercontent.com/23315232/119242052-c282da80-bb52-11eb-800f-e5cc4a4d050e.png">
 
 ## Step 5  
 Tested the nginx server with php by creating an info.php file in the document root for the project and adding lines to print info about the server
 ```
 sudo nano /var/www/projectLEMP/info.php
 ```
 ```
 <?php
phpinfo();
```
Result of visiting the page:

<img width="807" alt="php_info" src="https://user-images.githubusercontent.com/23315232/119242271-7c2e7b00-bb54-11eb-93b3-ca4a6d240309.png">

## Step 6
Retrieved data from the MySQL DB after creating an ```example_database``` and ```todo_list``` table. Results are shown below:

<img width="538" alt="db_access" src="https://user-images.githubusercontent.com/23315232/119242348-f828c300-bb54-11eb-8fa3-f12e1cd20436.png">

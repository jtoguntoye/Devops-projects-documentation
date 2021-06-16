# MEAN stack implementation

## Step 1- Installing NodeJs
- I updated and upgraded Ubuntu with the commands
```
sudo apt update
sudo apt upgrade
```
The version of Ubuntu OS installed on EC2 instance is shown below:

<img width="509" alt="ubuntu-version" src="https://user-images.githubusercontent.com/23315232/122067353-6c205900-cdeb-11eb-988e-80e5e0b9911f.png">

- Installed NodeJs with the command:
```
sudo apt install  -y nodejs
```
Node version installed: ``` v10.19.0 ```

## Step 2 Installing MongoDB
- Installed MongoDB and started mongoDB service with the commands:
```
sudo apt install -y mongodb
sudo service mongodb start
```
The Status of MongoDB service is shown below:

<img width="591" alt="mongodb_service_status" src="https://user-images.githubusercontent.com/23315232/122230937-faf7a900-ceb1-11eb-8cf5-0104cebfc963.png">

- Installed Node Package manager (npm). Version of npm installed: ``` 6.14.4 ```  

- Next, I installed the body-parser package using npm, created the ``` Books ``` directory and added the server.js file in it.

## Step 3- Installing Express Js and setting up routes to the server
- Next, Installed Express Js and mongoose (for setting up db schema) with the command:
```
sudo npm install express mongoose
```
- Next, created the ```apps``` directory and added the ```routes.js``` file in it.
- Also in the ```apps``` directory, created the ```models``` directory and added the ```books.js``` file with it contents

## Step 4 -Accessing the routes with AngularJs
- Created a ```public``` directory in the ```Books``` directory  and added a file named ```script.js``` in it 
- Also added the ```index.html``` file to the ```public``` directory with the required file content
- Next, I started the server by running 
 ```
 node server.js
 ```
 -To be able to access the server from the browser, I added a new inbound rule to the security group to allow TCP port 3300.
 
 The final result of accessing the server and adding new books is shown below:
 
 <img width="960" alt="result_page" src="https://user-images.githubusercontent.com/23315232/122253516-25eaf880-cec4-11eb-8565-802a20100050.png">




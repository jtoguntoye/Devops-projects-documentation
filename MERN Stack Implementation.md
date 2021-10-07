# MERN Stack Implementation

## Step 1 -Backend Configuration

Updated apt and upgraded Ubuntu using the following commands:
```
sudo apt update
sudo apt upgrade
```
Ubuntu version installed on EC2 server instance:

<img width="287" alt="ubuntu_upgrade" src="https://user-images.githubusercontent.com/23315232/119710839-c1092900-be56-11eb-995a-8c6e87cd9d6b.png">

Next, Installed Node.js on the server with the command
```
sudo apt-get install -y nodejs
```
Node version installed:

<img width="215" alt="node_v" src="https://user-images.githubusercontent.com/23315232/119711174-1fcea280-be57-11eb-9871-2b7bc4be5d26.png">

Next, Created a new directory for the project, initialized the project and installed Express JS with the commands:
```
mkdir Todo
npm init
npm install express
```
Next installed dontenv module, created the index.js file and added the required code.

Next, Created ```routes```  directory that will define the various endpoints that the To-do app will depend on. Also created an ```Api.js``` file inside the routes directory and added the required code.

Set up a mongoDB database and a collection using Mlab and added the connection string to the app's `.env` file. A screenshot of the DB on mlab is shown below:

<img width="938" alt="mongodb-cluster" src="https://user-images.githubusercontent.com/23315232/120111942-b955c700-c16b-11eb-88be-ae434790b395.png">

Next, tested the backend API using Postman.  Screenshot of Postman below:

<img width="940" alt="postman_screenshot" src="https://user-images.githubusercontent.com/23315232/120113605-1f921800-c173-11eb-98b7-9596ef2aa8e3.png">


## Creating Frontend
Result after adding the required frontend code is shown  below:

<img width="918" alt="TodoAPPResult" src="https://user-images.githubusercontent.com/23315232/120114215-fb840600-c175-11eb-8b55-8ca80f9ce3ef.png">



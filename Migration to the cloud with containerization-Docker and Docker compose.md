# Migration to the cloud with containerization-Docker and Docker compose

### Install Docker and prepare for migration to the cloud

- Install Docker engine locally on your machine or spin up an EC2 instance on AWS
 #### Note: if installed on a VM ensure you add the default user of the VM to the u docker group to allow the user run docker commands
  ```
   curl -fsSL https://get.docker.com -o get-docker.sh
  sh ./get-docker.sh
  usermod -aG docker ubuntu
  ```
  
  <img width="598" alt="docker_version" src="https://user-images.githubusercontent.com/23315232/146751061-7ce21d3a-d729-4aae-a30b-4384134fe1d0.png">

### MySQL in Container
- Pull MySQL Docker image from Docker Hub
  ```sh
  docker pull mysql/mysql-server:latest
  ```
    
  - Deploy the MySQL Container
  ```sh
  docker run --name mysql -e MYSQL_ROOT_PASSWORD=adminPassword -d mysql/mysql-server:latest
  ```
  
  You can change the **name** and **password** flags as desired.
  Run `docker ps -a` to see all your containers (running or stopped)
  
   <img width="849" alt="container_for_mysql-db" src="https://user-images.githubusercontent.com/23315232/146751194-1189b556-6f9f-4e38-a526-551568416f46.png">
   
  ### Connecting to the MySQL Docker container
- Connecting directly
  ```sh
  docker exec -it  mysql -u root -p
  ```
  
  ### Connecting to the MySQL Docker container
- Connecting directly
  ```sh
  docker exec -it  mysql -u root -p
  ```
  
  - Second approach
  - Create a network
    ```sh
    docker network create --subnet=172.18.0.0/24 tooling_app_network
    ```
  - Export an environment variable containing the root password setup earlier
    ```sh
    export MYSQL_PW=adminPassword
    ```
  - Run the container using the network created
    ```sh
    docker run --network tooling_app_network -h mysqlserverhost --name=mysql-server -e MYSQL_ROOT_PASSWORD=$MYSQL_PW  -d mysql/mysql-server:latest
    ```
    
  - Create a MySQL user using a script
  - Create a **create_user.sql** file and paste in the following
    ```
      CREATE USER 'admin'@'%' IDENTIFIED BY 'pass1234'; 
      GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%';
    ```
  - Run the script
    ```sh
    docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < ./create_user.sql
    ```

- Run the MySQL client container
  ```sh
  docker run --network tooling_app_network --name mysql-client -it --rm mysql mysql -h mysqlserverhost -u  -p 
  ```
  
  ### Prepare Database Schema
- Clone the Tooling app repository
  ```
  git clone https://github.com/darey-devops/tooling.git
  ```
- Export the location of the SQL file
  ```
  export tooling_db_schema=./tooling/html/tooling_db_schema.sql
  ```
- Use the script to create the database and prepare the schema
  ```
  docker exec -i mysql-server mysql -u root -p $MYSQL_PW < $tooling_db_schema
  
- Update *db_conn.php* file with database information
  ```
   $servername = "mysqlserverhost"; $username = ""; $password = ""; $dbname = "toolingdb"; 
   ```
   
   <img width="552" alt="db_conn file" src="https://user-images.githubusercontent.com/23315232/146754239-14323622-db31-4ff6-b342-30fdd98e36f2.png">
   
 - Run the Tooling app
  - First build the Docker image. Cd into the tooling folder, where the Dockerfile is and run
    ```
    docker build -t tooling:0.0.1 .
    ```
  - Run the container
    ```
    docker run --network tooling_app_network -p 8085:80 -it tooling:0.0.1
    ```

<img width="931" alt="tooling_app_running_in_a_container" src="https://user-images.githubusercontent.com/23315232/146754414-0bf9e4b3-8c82-4a08-9f16-1f3191bb3f12.png">


### Practice Task 1
### Implement a POC to migrate the PHP-Todo app into a containerized application.
### Part 1

- Clone the php-todo repo here: https://github.com/Anefu/php-todo.git 
- Write a Dockerfile for the TODO application
```
FROM php:7-apache
LABEL Joel joeloguntoye@gmail.com

# install , unzip extension and mysql client
RUN apt-get update --fix-missing && apt-get install -y \
    unzip \
    zip \
    curl 

# Install php dependencies for mysql
RUN docker-php-ext-install pdo_mysql mysqli

#copy over the virtualhost config file
COPY apache-config.conf /etc/apache2/sites-available/000-default.conf
COPY start-apache /usr/local/bin

RUN a2enmod rewrite

# Install php dependency - composer
RUN curl -sS https://getcomposer.org/installer |php && mv composer.phar /usr/local/bin/composer

# Copy application source files
COPY . /var/www/html

RUN chown -R www-data:www-data /var/www

EXPOSE 80

CMD ["start-apache"]
```

### Part 2
- Create an account on Docker Hub
- Create a Docker Hub repository
- - First run `docker login` and enter your docker credentials.
  - Then retag your image and push:
    ```
    docker tag <tag> <repo-name>/<tag>
    docker push <repo-name>/<tag>
    ```

<img width="937" alt="pushing_to_docker_hub" src="https://user-images.githubusercontent.com/23315232/146756396-d0002227-5465-425f-a59b-69858b5e5360.png">

<img width="955" alt="verifying_image_pushed_to_dockerhub" src="https://user-images.githubusercontent.com/23315232/146756429-52b2a4e3-35ff-4cf7-9667-adb512594b1e.png">

### part 3
- Set up a Jenkins server to be used to build  a Docker image
  -  Set up credentials for Jenkins to access Docker hub

  <img width="941" alt="setting_up_credentials_for_jenkins_to_Access_dockerhub" src="https://user-images.githubusercontent.com/23315232/146757416-1d0a37ff-e4d9-4bc3-bd6d-468de9e2aad4.png">
 
- Write a Jenkinsfile for Docker build and push to registry
```
pipeline {
 agent any

environment {
    DOCKERHUB_CREDENTIALS = credentials('joel-dockerhub')
  }
 stages {

    stage("Initial cleanup") {
          steps {
              dir("${WORKSPACE}") {
              deleteDir()
              }
          }
    }

    stage ('Checkout SCM') {
         steps {
             git branch: 'main', url: "https://github.com/jtoguntoye/docker-php-todo.git"
         }
    }

    stage ('Build docker image') {
        steps {
             sh "docker build -t joeltosin/todo-app:${env.BRANCH_NAME}-${env.BUILD_NUMBER} ."
        }
    }

    stage ('Login to Docker hub') {

        steps {
              sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
        
        }  
    }
    stage ('Docker push') {
        steps {
            sh "docker push joeltosin/todo-app:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
        }
    }
  }
  post 
  {
    always {
      sh 'docker logout'
    }
  }

}
```
- Connect the repo to Jenkins
- Create a multibranch pipeline
- Simulate a Docker push

<img width="1500" alt="pushing_to_docker_hub_From_jenkins" src="https://user-images.githubusercontent.com/23315232/146758990-9d31c5df-6e76-4e56-9304-995c214c275a.png">

- Verify the images can be found in the registry

<img width="955" alt="verifying_image_pushed_to_dockerhub" src="https://user-images.githubusercontent.com/23315232/146759201-0c17e19b-834b-428a-a0f2-84b0fc6e5238.png">

### Deployment with Docker Compose
- Install Docker Compose (https://docs.docker.com/compose/install/)
- Create a file, `tooling.yaml` and paste in the following block

```
version: '3.7'

services:
  tooling_frontend:
    build: .
    ports: 
      - 5000:80
    volumes:
      - tooling_frontend:/var/www/html  
    depends_on:
      - db
  
  db:
    image: mysql:5.7
    hostname: mysqlserverhost
    restart: always
    environment: 
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_ROOT_PASSWORD: ${ROOT_PASSWORD}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_USER_PASSWORD}
    volumes:
     - db:/var/lib/mysql

volumes:
  tooling_frontend:
  db:
```
- Run the command `docker-compose -f tooling.yaml up -d` to start the containers.
- Verify the containers are running `docker-compose -f tooling.yaml ps`

<img width="730" alt="docker-compose-app-running" src="https://user-images.githubusercontent.com/23315232/146759956-256d0f5e-23b1-4c0a-ae2e-db299bd4a73a.png">

### Practice Task 2
#### 1. Document understanding of the various fields in `tooling.yaml`

- version: "3.7" specifies the version of the docker-compose API used to write the docker-compose file
- services: defines configurations that are applied to containers when `docker-compose up` is run. 
  - tooling_frontend: specifies the name of the first service (container) to build .
  - build: tells docker-compose that it needs to build and image from a Dockerfile for this service.
  - ports: maps port 5000 on the instance to port 80 on the container
  - volumes: attaches a path on the host instance to any container created for the service
  - depends_on: Specifies the container the a given service depends on (tooling_frontend to db in this case)
  - db: defines the database service (can be any name)
  - image: specifies the image to use for the containers, if it isn't available on the instance, it is pulled from Docker Hub
  - restart: tells the container how frequently to restart
  - environment: used to pass environment variables required for the service running in the container
  
 
 #### 2. Update Jenkinsfile with a test stage
 
 ```
 stage ('Start the app')
    {
      steps {
        sh "docker compose up -d"
      }
    }

    stage ('Test the endpoint and push image to dockerhub if sucessful')
    {
      steps {
        script{
          while (true) {
            def response = httpRequest 'http://localhost:8200'
            if(response == 200) {
              sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
              sh "docker push joeltosin/todo-app:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
            }
            break
          }
        }

      }
    }
 ```
 
  The "Test endpoint" stage uses httpRequest to check if the app can be reached after running `docker-compose up -d`
  
  - A post stage to always clean up the server and delete images on the Jenkins server
  ```
  post 
  {
    always {
      sh "docker-compose down"
      sh "docker system prune -af"
      sh 'docker logout'
    }
  }
  ```
 
 

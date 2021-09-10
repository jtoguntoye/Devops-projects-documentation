## Continuous integration with Jenkins | Ansible | Artifactory | Sonarqube | PHP

## Step 1: Simulating a typical CI/CD pipeline for a PHP based application

### 1.1 SetUp
- Ansible inventory will look like this
```
├── ci
├── dev
├── pentest
├── pre-prod
├── prod
├── sit
└── uat
```

CI inventory file
```
[jenkins]
<Jenkins-Private-IP-Address>

[nginx]
<Nginx-Private-IP-Address>

[sonarqube]
<SonarQube-Private-IP-Address>

[artifact_repository]
<Artifact_repository-Private-IP-Address>
```

Dev
```
[tooling]
<Tooling-Web-Server-Private-IP-Address>

[todo]
<Todo-Web-Server-Private-IP-Address>

[nginx]
<Nginx-Private-IP-Address>

[db:vars]
ansible_user=ec2-user
ansible_python_interpreter=/usr/bin/python

[db]
<DB-Server-Private-IP-Address>
```

Pen test environment
```

[pentest:children]
pentest-todo
pentest-tooling

[pentest-todo]
<Pentest-for-Todo-Private-IP-Address>

[pentest-tooling]
<Pentest-for-Tooling-Private-IP-Address>
```

- In th pentest inventory file, we want to have a group called pentest which covers Ansible execution against both pentest-todo and pentest-tooling simultaneously. But at the same time, we want the flexibility to run specific Ansible tasks against an individual group.

## Step 1.2: Ansible Roles for CI environment
- 1. Sonarqube role: SonarQube is an open-source platform developed by SonarSource for continuous inspection of code quality, it is used to perform automatic reviews with static analysis of code to detect bugs, code smells, and security vulnerabilities
- 2. Artifactory role: Artifactory is a product by JFrog that serves as a binary repository manager. The binary repository is a natural extension to the source code repository, in that the outcome of your build process is stored.

## Step 1.3 Configure Jenkins for Ansible Deployment
- In previous projects we have been launching Ansible playbooks from command line. Now, we'll setup Jenkins to be able to run ansible playbooks from Jenkins UI.
  
- Navigate to Jenkins UI and install Blue Ocean plugin

- Click Open Blue Ocean from the left pane

- Once you're in the Blue Ocean UI, click Create Pipeline

- Select GitHub

- On the Connect to GitHub step, click 'Create an access token here' to create your access token

- Type in the token name of your choice, leave everything as is and click Generate token

- Copy the generated token and paste in the field provided in Blue Ocean and click connect

- Select your organization (typically your GitHub repository name)

- Select the repository to create the pipeline from

<img width="946" alt="new_pipeline_created_for_ansible_github_repo" src="https://user-images.githubusercontent.com/23315232/132571863-f1ae22c7-69f6-4acf-91b0-4b3b75af6d4d.png">


- Blue Ocean would take you to where to create a pipeline, since we are not doing this now, click Administration from the top bar to exit Blue Ocean.

### Create `JenkinsFile`
-  A Jenkinsfile is a text file that contains the definition of a Jenkins Pipeline and is checked into source control

- Inside the Ansible project, create a new directory deploy and start a new file Jenkinsfile inside the directory.
- Add the following code snippet to the `jenkinsFile`

```
pipeline {
agent any
    stages {
        stage('Build') {
            steps {
                script {
                    sh 'echo "Building Stage"'
                }
            }
        }
    }
}
```
- Next, in the ansible project in jenkins select configure. Scroll down to Build Configuration section and specify the location of the Jenkinsfile at deploy/Jenkinsfile

<img width="730" alt="specifying_the_location_of_the_jenkinsfile_in_jenkins_Server" src="https://user-images.githubusercontent.com/23315232/132573951-98b75794-72e2-455a-825d-bdfde579dcbc.png"> 

- Build the project in jenkins and view in blue ocean UI

- Notice that this pipeline is a multibranch one. This means, if there were more than one branch in GitHub, Jenkins would have scanned the repository to discover them all and we would have been able to trigger a build for each branch

- Create a new git branch and name it `feature/jenkinspipeline-stages`
- Currently we only have the Build stage. Let us add another stage called Test. Paste the code snippet below and push the new changes to GitHub.

```
pipeline {
    agent any

  stages {
    stage('Build') {
      steps {
        script {
          sh 'echo "Building Stage"'
        }
      }
    }

    stage('Test') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }
    }
}
```
- To make your new branch show up in Jenkins, we need to tell Jenkins to scan the repository
-Refresh the page and both branches will start building automatically. You can go into Blue Ocean and see both branches there too. 
- Below is a view of the pipeline for the `feature/jenkinspipeline-stages` branch

<img width="921" alt="visualizing_pipeline_build_for_another_branch_in_blue_ocean_ui" src="https://user-images.githubusercontent.com/23315232/132574802-3283d71f-af7a-4586-9a15-c5e0eb6ad358.png">

```
Quick task:
1. Create a pull request to merge the latest code into the `main branch`
2. After merging the `PR`, go back into your terminal and switch into the `main` branch.
3. Pull the latest change.
4. Create a new branch, add more stages into the Jenkins file to simulate below phases. (Just add an `echo` command like we have in `build` and `test` stages)
   1. Package 
   2. Deploy 
   3. Clean up
5. Verify in Blue Ocean that all the stages are working, then merge your feature branch to the main branch
```
- Eventually, your main branch should have a successful pipeline like this in blue ocean

<img width="925" alt="master_branch_build_pipeline_After_merging_feature_branch" src="https://user-images.githubusercontent.com/23315232/132575068-fc22b479-8867-46ca-a09f-8fcbf86335de.png">

### Run Ansible playbooks from Jenkins
- Install Ansible on Jenkins
- Install Ansible plugin in Jenkins UI
- Create jenkinsFile from scratch. 
 #### NOTES: 
      - Ensure that the git module in `Jenkinsfile` is checking out SCM to `main` branch instead of `master`. (GitHub has discontinued the use of `Master` due to Black           Lives Matter)
      - Jenkins needs to export the ANSIBLE_CONFIG environment variable. You can put the .ansible.cfg file alongside Jenkinsfile in the deploy directory. This way, anyone         can easily identify that everything in there relates to deployment. Then, using the Pipeline Syntax tool in Ansible, generate the syntax to create environment             variables to set
      - Ensure that you start the Jenkinsfile with a clean up step to always delete the previous workspace before running a new one. This ensures that Jenkins downloads           the latest code from Github
      
```
pipeline {
    agent any

     options {
        // This is required if you want to clean before build
        skipDefaultCheckout(true)
    }
    environment {
      ANSIBLE_CONFIG="${WORKSPACE}/deploy/ansible.cfg"
    }
    stages {
         stage("Initial cleanup") {
          steps {
            // Clean before build
                cleanWs()
          }
        }
        stage ('SCM checkout') {
          steps {
            git(branch: 'master', url: 'https://github.com/jtoguntoye/ansible-config-mgt')
            }
        }
        stage ('Exexcute playbook') {
            steps {
                ansiblePlaybook credentialsId: 'latest-key', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory/dev', playbook: 'playbooks/site.yml'
            }
        }

        stage('Clean Workspace after build'){
           steps{
           cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, deleteDirs: true)
           }
       }
    }
}
```


<img width="682" alt="console_output_after_running_ansible_playbook_from_jenkins" src="https://user-images.githubusercontent.com/23315232/132630664-4f3df1ea-bda0-4db8-ae84-8b4c15f97a0d.png">

### Parameterizing Jenkinsfile For Ansible Deployment
- In the above JenkinsFile, we are deploying the ansible playbook to the dev environment. If we need to deploy to other environments (say pentest, sit environments), we     need to pass parameters to the jenkinFile

1- Update SIT inventory with new servers
2- Update Jenkinsfile to introduce parameterization. Below is just one parameter. It has a default value in case if no value is specified at execution. It also has a  description so that everyone is aware of its purpose.

```
pipeline {
    agent any

    parameters {
      string(name: 'inventory', defaultValue: 'dev',  description: 'This is the inventory file for the environment to deploy configuration')
    }
...
```

- The entire jenkinsFile now looks like this: 

```
pipeline {
    agent any

     parameters {
       string(name: 'inventory_file', defaultValue: '', description: 'This is the inventory file for the environment to deploy configuration')
     }
     options {
        // This is required if you want to clean before build
        skipDefaultCheckout(true)
    }
    environment {
      ANSIBLE_CONFIG="${WORKSPACE}/deploy/ansible.cfg"
    }
    stages {
         stage("Initial cleanup") {
          steps {
            // Clean before build
                cleanWs()
          }
        }
        stage ('SCM checkout') {
          steps {
            git(branch: 'master', url: 'https://github.com/jtoguntoye/ansible-config-mgt')
            }
        }
        stage ('Exexcute playbook') {
            steps {
                ansiblePlaybook credentialsId: 'latest-key', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory/${inventory_file}', playbook: 'playbooks/site.yml'
            }
        }

        stage('Clean Workspace after build'){
           steps{
           cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, deleteDirs: true)
           }
       }
    }
}
```
- When you execute the pipeline now, you get prompted to enter the inventory file to be used

<img width="923" alt="jenkins_requesting_inventory_parameter_for_ansible_deployment" src="https://user-images.githubusercontent.com/23315232/132812551-0a7b95af-bd2d-4abc-8a14-bb0d103b7a05.png">


## Step 2: CI/CD Pipeline for TODO application
- Our goal here is to deploy the application onto servers directly from Artifactory server rather than from `git`

### Step 2.1 Configure Artifactory
- Launch a new server instance to be used as artifactory server. Note: Choose an EC2 instance size above micro (use medium size)
- Install artifactory on the server
```
sudo apt update
wget https://releases.jfrog.io/artifactory/artifactory-rpms/artifactory-rpms.repo -O jfrog-artifactory-rpms.repo
sudo mv jfrog-artifactory-rpms.repo /etc/yum.repos.d/
sudo yum update && sudo yum install jfrog-artifactory-oss
ln -s /etc/opt/jfrog/artifactory/ /opt/jfrog/artifactory
```

<img width="489" alt="artifactory_installed_on_Server" src="https://user-images.githubusercontent.com/23315232/132818911-47ecc340-6948-4e1a-ab20-302cf7075760.png">

### Step 2.2 Prepare Jenkins

- Fork the php-todo repository  below to your github account
```
https://github.com/darey-devops/php-todo.git
```
- On you Jenkins server, install PHP, its dependencies(Feel free to do this manually at first, then update your Ansible accordingly later)

```
sudo apt install -y zip libapache2-mod-php php7.4-fpm phploc php-{xml,bcmath,bz2,intl,gd,mbstring,mysql,zip,xdebug}
```
- Install PHP Composer manually:
```
   php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
   
   php -r "if (hash_file('sha384', 'composer-setup.php') === '756890a4488ce9024fc62c56153228907f1545c228516cbf63f885e036d37e9a59d27d63f46af1d4d07ee0f76181c7d3') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
   
   php composer-setup.php
   
   mv composer.phar /usr/local/bin/composer
```
- Configure the php.ini file, you can get the path to the file by running

- After getting the path to the file, enter the file and paste in:
```
xdebug.mode = coverage
```
- Install nodejs and npm
```
sudo apt-get update -y
sudo apt-get install nodejs -y
sudo apt install npm -y
```
- Install typescript using node package manager(npm). Only root user can install typecript.
```
su
npm install -g typescript
```
- Restart php7.4-fpm
```
sudo systemmctl restart php7.4-fpm
```
- Next, install Tthe following plugins on Jenkins UI

    - Plot plugin: to display tests reports and code coverage information
    - Artifactory plugin: to easily deploy artifacts to Artifactory server

- Go to the artifactory URL(http://artifactory-server-ip:8082) and create a local generic repository named php-todo(Default username and password is admin. After logging in, change the password)

- Configure Artifactory in Jenkins UI
- Click Manage Jenkins, click Configure System
- Scroll down to JFrog, click Add Artifactory Server
- Enter the Server ID
- Enter the URL as:

```
http://<artifactory-server-ip>:8082/artifactory
```
- Enter the Default Deployer Credentials(the username and the changed password of artifactory)

<img width="716" alt="configure_Artifactory_ip_url_and_Server_credentials_in_jenkins" src="https://user-images.githubusercontent.com/23315232/132847281-f9a22b8b-6c48-421d-b659-57549c295b4c.png">

## Step 2.3 Integrate Artifactory Repository with Jenkins
- On Jenkins server, install mysql-client

- Create a dummy Jenkinsfile in php-todo repo

- In Blue Ocean, create multibranch php-todo pipeline(follow the previous steps earlier)

- Spin up an instance for a database, install and configure mysql-server( This is where data pertaining to the artifactory repository will be stored).

- Create a database and user on the database server

```
CREATE DATABASE homestead;
CREATE USER 'homestead'@'%' IDENTIFIED BY 'sePret^i';
GRANT ALL PRIVILEGES ON * . * TO 'homestead'@'%';
FLUSH PRIVILEGES;
```
- Check if the database created on the database server can be reached from the Jenkins server. On the jenkins server, run command:
``` mysql -u <DB_user> -h <DB-private-ip-address> -p
```
- Next update the `.env.sample` file in your php-todo repo to connect to the external DB

<img width="588" alt="setting_artifacctory_external_db_ip_address_in_the_repos_env_file" src="https://user-images.githubusercontent.com/23315232/132848447-e84760cf-bad3-443a-817e-9e8d1dbbd701.png">

- Update Jenkinsfile with proper configuration
```
pipeline {
  agent any

  stages {

   stage("Initial cleanup") {
        steps {
          dir("${WORKSPACE}") {
            deleteDir()
          }
        }
      }

  stage('Checkout SCM') {
    steps {
          git branch: 'main', url: 'https://github.com/darey-devops/php-todo.git'
    }
  }

  stage('Prepare Dependencies') {
    steps {
           sh 'mv .env.sample .env'
           sh 'composer install'
           sh 'php artisan migrate'
           sh 'php artisan db:seed'
           sh 'php artisan key:generate'
        }
      }
    }
  }
```
- In the above jenkinsFile:
  - Composer is used by PHP to install all the dependent libraries used by the application
  - php artisan uses the .env file to setup the required database objects - (After successful run of this step, login to the database, run show tables and you will see the tables being created for you)

<img width="894" alt="successfully_built_main_branch_for_todo_repo" src="https://user-images.githubusercontent.com/23315232/132849991-3ddea335-0b45-4843-a6ee-a702b787e61e.png">

  
  - Next, update the Jenkinsfile to include Unit tests step
  ```
  stage('Execute Unit Tests') {
      steps {
             sh './vendor/bin/phpunit'
      } 
  ```

 ## Step 2.4 Code Quality analysis
 - For PHP the most commonly tool used for code quality analysis is phploc
 - The data produced by phploc can be ploted onto graphs in Jenkins.
 -Add the code analysis step in Jenkinsfile. The output of the data will be saved in build/logs/phploc.csv file.
 ```
 stage ('Code analysis') {
            steps {
                sh 'phploc app/ --log-csv build/logs/phploc.csv'
            }
        }
 ```
 - Plot the data using plot Jenkins plugin.
 ```
  stage('Plot Code Coverage Report') {
      steps {

            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)                          ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'A - Lines of code', yaxis: 'Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Directories,Files,Namespaces', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'B - Structures Containers', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Average Class Length (LLOC),Average Method Length (LLOC),Average Function Length (LLOC)', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'C - Average Length', yaxis: 'Average Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Cyclomatic Complexity / Lines of Code,Cyclomatic Complexity / Number of Methods ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'D - Relative Cyclomatic Complexity', yaxis: 'Cyclomatic Complexity by Structure'      
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Classes,Abstract Classes,Concrete Classes', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'E - Types of Classes', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Methods,Non-Static Methods,Static Methods,Public Methods,Non-Public Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'F - Types of Methods', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Constants,Global Constants,Class Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'G - Types of Constants', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Test Classes,Test Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'I - Testing', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Logical Lines of Code (LLOC),Classes Length (LLOC),Functions Length (LLOC),LLOC outside functions or classes ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'AB - Code Structure by Logical Lines of Code', yaxis: 'Logical Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Functions,Named Functions,Anonymous Functions', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'H - Types of Functions', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Interfaces,Traits,Classes,Methods,Functions,Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'BB - Structure Objects', yaxis: 'Count'


      }
    }
 ```
 - On running, the pipeline, a new option `plot` appears on Jenkins UI from where you can view the analysis chart for the analysed code
 ```
 <img width="932" alt="plot_plugin_result_for_phploc_code_Analysis_stage" src="https://user-images.githubusercontent.com/23315232/132850794-2f449965-4b17-441c-9069-0440e48fae34.png">
 
 -  Next, Bundle the application code for into an artifact (archived package) upload to Artifactory server
 ``` 
 - stage ('Package Artifact') {
    steps {
            sh 'zip -qr ${WORKSPACE}/php-todo.zip ${WORKSPACE}/*'
}

- Publish the resulted artifact into Artifactory
```
stage ('Deploy Artifact to Artifactory') {
          steps {
            script {
                 def server = Artifactory.server 'artifactory-server'
                 def uploadSpec = """{
                    "files": [
                      {
                       "pattern": "php-todo.zip",
                       "target": "joel/php-todo",
                       "props": "type=zip;status=ready"
                       }
                    ]
                 }"""
                 server.upload spec: uploadSpec
               }
            }
        }
```
- Deploy the application to the dev environment by launching Ansible pipeline

```
tage ('Deploy to Dev Environment') {
    steps {
    build job: 'ansible-project/main', parameters: [[$class: 'StringParameterValue', name: 'env', value: 'dev']], propagate: false, wait: true
    }
  }
```
Note:
The build job used in this step tells Jenkins to start another job. In this case it is the ansible-project job, and we are targeting the main branch. Hence, we have ansible-project/main. Since the Ansible project requires parameters to be passed in, we have included this by specifying the parameters section. The name of the parameter is env and its value is dev. Meaning, deploy to the Development environment.

<img width="805" alt="succesfully_deploying_artifact_to_artifactory_Server_And_then_running_ansible_job" src="https://user-images.githubusercontent.com/23315232/132851412-7aa72466-5b19-4818-a7c7-3b2d498d25f3.png">

## Step 3 - Configure Sonarqube
- To ensure only code that meets corporate customer's requirements is deployed to Artifactory, we'll set up a quality gate.
- SonarQube is a tool that can be used to create quality gates for software projects, and the ultimate goal is to be able to ship only quality software code
- In this project we will use predefined Quality Gates (also known as The Sonar Way). Software testers and developers would normally work with project leads and architects to create custom quality gates

### Step 3.1 Install SonarQube on Ubuntu 20.04 With PostgreSQL as Backend Database
- Tune Linux kernel
```
sudo sysctl -w vm.max_map_count=262144
sudo sysctl -w fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
```
- Open the /etc/security/limits.conf file and append the following.
```
sonarqube   -   nofile   65536
sonarqube   -   nproc    4096
```
- To enable persistence after reboot, open the /etc/sysctl.conf and append the following.
```
vm.max_map_count=262144
fs.file-max=65536 
```
### Step 3.2: Update system packages and Install Java(sonarqube is java-based) and other required packages
` Update and upgrade packages
```
sudo apt-get update -y
sudo apt-get upgrade -y
```
- Install wget and unzip packages
```
sudo apt-get install wget unzip -y
```
- Install OpenJDK and Java Runtime Environment (JRE)
```
sudo apt-get install openjdk-11-jdk -y
sudo apt-get install openjdk-11-jre -y
```
- To set default JDK or switch to OpenJDK, run the command,
```sudo update-alternatives --config java```

- Verify JAVA is installed by running:
```java -version```
### Step 3.3: Install and Setup PostgreSQL 10 Database for SonarQube
- Go to the security group and open port 5432 for postgreSQL on Sonarqube Instance

- Add PostgreSQL to repo list
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
```
- Download PostgreSQL software
```
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
```
- Install PostgreSQL
```
sudo apt-get -y install postgresql postgresql-contrib
```
- Start and enable PostgreSQL Database service
```
sudo systemctl start postgresql
sudo systemctl enable postgresql
```
- Change the default password for postgres user (to any password you can easily remember)
```sudo passwd postgres```
- Then switch to postgres user
```su - postgres```

- Create a new user for SonarQube

``createuser sonar``
-  Create Sonar DB and user

  - Switch to PostgreSQL shell
  ``psql```
  - Set up encrypted password for newly created user
  ```ALTER USER sonar WITH ENCRYPTED password 'sonar';```
  - Create a DB for SonarQube
  ```CREATE DATABASE sonarqube OWNER sonar;```
  - Grant required privileges
  ```grant all privileges on DATABASE sonarqube to sonar;```
  `Exit the shell
```\q```
- Switch back to initial user
  ```exit```
  
  ### Step 3.4: Install and configure SonarQube
- Download temporary files to /tmp folder
  ```
  cd /tmp && sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.9.3.zip
  ```
- Unzip archive to the /opt directory and move to /opt/sonarqube
  ```
  sudo unzip sonarqube-7.9.3.zip -d /opt
  sudo mv /opt/sonarqube-7.9.3 /opt/sonarqube
  ```
    
   
- Configure SonarQube
Since Sonarqube cannot be run as root user, we have to create a **sonar** user to start the service with.
  - Create a group sonar
    ```
    sudo groupadd sonar
    ```
  - Add a **sonar** user with control over /opt/sonarqube
    ```
    sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar 
    sudo chown sonar:sonar /opt/sonarqube -R
    ```
  - Open and edit SonarQube configuration file
    
    ```
    sudo vim /opt/sonarqube/conf/sonar.properties
    ```
    
    Then find and edit the following lines
  
    ```
    sonar.jdbc.username=sonar
    sonar.jdbc.password=sonar
    ```
    Append this line
    ```
    sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
    ```
    
    - Enter the sonar script file. Find and set RUN_AS_USER to be equals to sonar
    
    ```
    sudo nano /opt/sonarqube/bin/linux-x86-64/sonar.sh
    ```
    ```
    # If specified, the Wrapper will be run as the specified user.

    # IMPORTANT - Make sure that the user has the required privileges to write

    #  the PID file and wrapper.log files.  Failure to be able to write the log

    #  file will cause the Wrapper to exit without any way to write out an error

    #  message.

    # NOTE - This will set the user which is used to run the Wrapper as well as

    #  the JVM and is not useful in situations where a privileged resource or

    #  port needs to be allocated prior to the user being changed.

     RUN_AS_USER=sonar
    ```
   
   - Add sonar user to sudoers file
    - First set the user's password to something you can easily remember
      ```
      sudo passwd sonar
      ```
    - Then run the command to add sonar to sudo group
      ```
      sudo usermod -a -G sudo sonar
      ```
    - To check if it has been added, run command
    ```
    groups sonar
    ```
    
  - Switch to the sonar user, switch to script directory and start the script
    ```
    su - sonar
    cd /opt/sonarqube/bin/linux-x86-64/
    ./sonar.sh start
    ```
    Output:
    ```
    Starting SonarQube...

    Started SonarQube
    ```
  - Check SonarQube logs
    ```
    tail /opt/sonarqube/logs/sonar.log
    ```
    
    <img width="856" alt="output_of_sonarqube_logs" src="https://user-images.githubusercontent.com/23315232/132855459-5f71fe85-5303-4337-82a0-508d31166369.png">

- Configure systemd to manage SonarQube service
    - Stop the sonar service
      ```
      cd /opt/sonarqube/bin/linux-x86-64/
      ./sonar.sh stop
      ```
    - Create systemd service file
      ```
      sudo vim /etc/systemd/system/sonar.service
      ```
      Paste in the following lines:
      ```
      [Unit]
      Description=SonarQube service
      After=syslog.target network.target

      [Service]
      Type=forking

      ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
      ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

      User=sonar
      Group=sonar
      Restart=always

      LimitNOFILE=65536
      LimitNPROC=4096

      [Install]
      WantedBy=multi-user.target
      ```
       Save and exit
    - Use systemd to manage the service
      ```
      sudo systemctl start sonar
      sudo systemctl enable sonar
      sudo systemctl status sonar
      ```
      
      <img width="939" alt="sonar_service_running_After_creating_systemd_file" src="https://user-images.githubusercontent.com/23315232/132854698-0f142b87-5850-48b2-80e6-1420db0c9ad2.png">
      
### Step 3.5: Access Sonarqube
- Access the SonarQube by going to http://sonarqube-public-ip-address:9000, use **admin** as your username and password

<img width="933" alt="login_to_sonarqube_Server" src="https://user-images.githubusercontent.com/23315232/132855924-6bd070ba-89f2-4cf9-8dcb-6e4be6ae40fd.png">
    
 **Note**: Do not forget to open ports 9000 and 5432 on the security group.
 
 ### Step 3.6: Configure SonarQube and Jenkins for Quality Gate
- Install SonarQube plugin in Jenkins UI
- Navigate to Manage Jenkins > Configure System and add a SonarQube server
  - In the name field, enter **sonarqube**
  - For server URL, enter the IP of your SonarQube instance
- Generate authentication tokens to use in the Jenkins UI
  - In SonarQube UI, navigate to User > My Account > Security > Generate tokens
  - Type in the name of the token and click Generate

- Configure SonarQube webhook for Jenkins
  -  Navigate to Administration > Configuration > Webhooks > Create
  - Enter the name
  - Enter URL as http://jenkins-server-ip:8080/sonar-webhook/

![configure_sonarqube_in_jenkins](https://user-images.githubusercontent.com/23315232/132856625-81dc4958-372c-4850-a7b8-a4068180ea2b.jpg)

- Setup SonarScanner for Jenkins
  - In Jenkins UI, go to Manage Jenkins > Global Tool Configuration
  - Look for SonarQube Scanner
  - Click 'Add SonarQube Scanner' and enter the scanner name as 'SonarQubeScanner'
  - Check the 'Install automatically' box
  - Select 'Install from Maven Central'
  
  ![configure_Sonar)_Scanner_Ansible](https://user-images.githubusercontent.com/23315232/132856815-1666b023-406b-4905-b559-fa4d3c136d0f.jpg)

-  Save and run the pipeline to install the scanner. An error will be generated but it will also create "/var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/conf/sonar-scanner.properties" directory

- Edit sonar-scanner.properties file
  ```
  sudo vim /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/conf/sonar-scanner.properties
  ```
  Paste in the following lines:
  ```
  sonar.host.url=http://<SonarQube-Server-IP-address>:9000
  sonar.projectKey=php-todo
  #----- Default source code encoding
  sonar.sourceEncoding=UTF-8
  sonar.php.exclusions=**/vendor/**
  sonar.php.coverage.reportPaths=build/logs/clover.xml
  sonar.php.tests.reportPath=build/logs/junit.xml
  ```
  
  - To further examine the configuration of the scanner tool on the Jenkins server - navigate into the tools directory
```
cd /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/bin. 
```
- List the content to see the scanner tool *sonar-scanner* with command "ls -latr". That is what we are calling in the pipeline script.

<img width="858" alt="listing_content_of_sonarScanner_bin_directory_showing_the_file_ran_in_the_jenkins_pipeline" src="https://user-images.githubusercontent.com/23315232/132856975-85250216-d0b4-4c49-9fef-bf67fd23e3be.png">

- - Add the following build stage for Quality Gate
  ```
  stage('SonarQube Quality Gate') {
      environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner"
            }

        }
    }
  ```
  
  - After running the pipeline, the output is shown below:
 
 <img width="937" alt="successfully_built_main_branch_after_adding_sonarqube_gate_for_todo_repo" src="https://user-images.githubusercontent.com/23315232/132857534-03b85b90-f780-4a9c-af0d-a0ed8ad63fd9.png">
 
- Also on the sonarqube UI, the result of the sonarqube quality check is shown below:

 <img width="947" alt="php-todo-project-code-analysis-shown-in-sonarqube" src="https://user-images.githubusercontent.com/23315232/132858174-ca078221-0fb4-4206-a44e-bcf2f6503075.png">   

### Step 3.7: Conditionally Deploy to Higher Environments
- Depending on the release strategy adopted for a project, we can use different branches to control how releases are done
- Assuming a basic gitflow implementation restricts only the ***develop*** branch to deploy code to Integration environment like ***sit***
  -  We will include a **when** condition to execute the Quality Gate stage only when the running branch is `develop`, `hotfix`, `release`, `main`
  -  Then we add a timeout step to wait for SonarQube to complete analysis and successfully finish the pipeline only when code quality is acceptable.
  -   The updated sonarqube gate analysis stage in the jenkinsFile is shown below:
 
 ```
  stage('SonarQube Quality Gate') {
      when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP"}
        environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner -Dproject.settings=sonar-project.properties"
            }
            timeout(time: 1, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
            }
        }
    }
  ```
  
  - To test, create different branches and push to GitHub. You will realise that only branches other than develop, hotfix, release, main, or master will be able to deploy the code.
  
  - Below is the pipeline build for **main** branch
  
  <img width="923" alt="project_not_Deployed_to_dev_environment_because_quality_Gate_did_not_pass" src="https://user-images.githubusercontent.com/23315232/132859193-0b180e86-2d8a-4384-a44a-d595241870f0.png">
  
  - Below is the pipeline build for  a newlyl create **new-feature** branch which does not pass through a quality gate
  
  <img width="947" alt="new-feature-branch-is-deployed-to-dev-environment-since-it-is-not-covered-in-quality-gate" src="https://user-images.githubusercontent.com/23315232/132859367-d43728b7-49f2-46d5-bea5-186dfcb0bc58.png">
  
## Step 4: Configure Jenkins slave servers
Setting up slave servers will help distribute the job workload in event of high wowrkloads. The Jenkins slave will execute jobs dispatched to it by the master server
- Spin up a new EC2 Instance
  - Install all the neccessary software packages just like you did with the master(bastion) server including configuring sonar-scanner.properties file
  
- On the main Jenkins server
  - Navigate to Manage Jenkins > Manage Nodes
  - Click New Node
  - Enter name of the node and click the 'Permanent Agent' button and click the OK button
  - Fill in the remote root directory as /home/ubuntu
  - Set 'Host' value as the Public-IP of the slave node
  - For Launch Method, select Launch Agents via SSH
  - Add SSH with username and private key credentials with username as ubuntu and private key as the private key of the master node
  - For Host Key Verification Strategy, select Manually trusted key validation strategy
  - Click Save
  
  <img width="957" alt="setting_up_jenkins_slave_node_on_master" src="https://user-images.githubusercontent.com/23315232/132859845-8ac87473-e6e6-4ee3-a000-eac418237a85.png">
  
  ## Step 5: Configure GitHub WebHook for Automatic Build of Pushed Code
    
 To automatically run the pipeline when there is a code push, we will configure github webhook
 
 <img width="938" alt="github_webhook_for_php-todo-repo" src="https://user-images.githubusercontent.com/23315232/132860270-97f67791-288f-42f2-9c69-38b2c7158aed.png">
 
 ## Step 6: Deploy to all Environments
 To deploy to all environemnts, we will configure the jenkinsfile in the config-mgt-ansible folder in two key places.
- We leave the *defaultValue* in parameters blank like this: **defaultValue: ''**
- In the Execute Ansible Stage, we will update the *inventory* like this: inventory: **'inventory/${inventory_file}'**


Pipeline Demo video link: https://drive.google.com/file/d/1sDSFIvD0M0DlKbp6jJvDEH48VMsbUrrM/view?usp=sharing


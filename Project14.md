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

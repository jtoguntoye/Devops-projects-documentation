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

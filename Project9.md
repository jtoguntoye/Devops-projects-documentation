# Continuous Integration Pipeline for Tooling Website

## Task:
Configure a job to automatically deploy source code changes of Tooling website (from project 8) to NFS server

## Updated Architecture

<img width="637" alt="project_architecture" src="https://user-images.githubusercontent.com/23315232/125121478-e80c7900-e0eb-11eb-88b2-ce88a8922fca.png">
Source: Darey.io

## Install and configure Jenkins server
- Launch an AWS EC2 instance on Ubuntu 20.04 LTS and name it "Jenkins"
- Install JDK (Java Development Kit) since Jenkins is a Java based application
```
sudo apt update
sudo apt install default-jdk-headless
```
Java version installed on the Jenkins server is shown below:

<img width="482" alt="java_version_installed" src="https://user-images.githubusercontent.com/23315232/125182242-4167cb80-e204-11eb-92fa-6c8e484bd839.png">

- Install Jenkins 
```
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \ /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt-get install jenkins
```
- confirm Jenkins status:
```
sudo systemctl status jenkins
```
<img width="478" alt="jenkins_service_status" src="https://user-images.githubusercontent.com/23315232/125182315-f7331a00-e204-11eb-9983-192938f289db.png">

- Open default TCP port 8080 for Jenkins on the server
- Perform initial Jenkins setup by accessing Jenkins server at `http://<Jenkins-Server-Public-IP-Address-or-Public-DNS-Name>:8080`

<img width="908" alt="unlock_jenkins" src="https://user-images.githubusercontent.com/23315232/125183129-e76b0400-e20b-11eb-9f39-089f0b6460eb.png">

- Login with the default Admin password for jenkins. The password can  be obtained from the file ``` /var/lib/jenkins/secrets/initialAdminPassword```
- Next, install the recommended plugins

## Configure Jenkins to retrieve source code from github using webhooks
- Enable webhooks in Github repository

<img width="832" alt="creating_webhook" src="https://user-images.githubusercontent.com/23315232/125183330-ae339380-e20d-11eb-9355-2d2e915a6535.png">

- Create a new freestyle project in Jenkins
- Connect the new project to the Github repository for the tooling website 

<img width="915" alt="source_code_management_jenkins" src="https://user-images.githubusercontent.com/23315232/125183588-9f4de080-e20f-11eb-82b5-1a06d0900162.png">

- Save the configuration and run the build

<img width="484" alt="building_project_in_jenkins" src="https://user-images.githubusercontent.com/23315232/125183612-cefce880-e20f-11eb-976e-374a3fdb642a.png">

- Configure triggering the build from github webhook (so that the build is started automatically everytime a new commit is pushed to the github repository)

<img width="597" alt="configuring_github_hook_trigger_for_the_jenkins_job" src="https://user-images.githubusercontent.com/23315232/125183643-3450d980-e210-11eb-83a9-cd388dfa5a6b.png">

- Configure post-build action to archive all the files resulting from the build (i.e artifacts)

<img width="586" alt="adding_post_build_Actions_to_archive_all_the_built_files" src="https://user-images.githubusercontent.com/23315232/125183687-7bd76580-e210-11eb-88f2-0a09e5240ba3.png">

- Making changes to the tooling website repo on Github will now trigger a build and a post-build action of creating an artifact

<img width="764" alt="new_build_in_jenkins_after_pushing_new_commit_to_master_branch" src="https://user-images.githubusercontent.com/23315232/125183721-c822a580-e210-11eb-92e8-c2fa28d76467.png">

By default artifacts are stored in  ``` /var/lib/jenkins/jobs/tooling_github/builds/<build_number>/archive/ ```

<img width="482" alt="archived_artifact_for_build_2" src="https://user-images.githubusercontent.com/23315232/125183754-f0120900-e210-11eb-8616-8bff2243754c.png">

## Configure jenkins to copy files to NFS server via SSH

Configure the Jenkins server to copy the artifacts to the NFS server to /mnt/apps directory
- Install "Publish over SSH" plugin

<img width="921" alt="publish_over_ssh_plugin_installed_on_jenkins_server" src="https://user-images.githubusercontent.com/23315232/125183940-0ff5fc80-e212-11eb-850e-9adf873c7e68.png">

- Configure the Jenkins job to copy artifacts over to NFS server

<img width="682" alt="configuring_jenkins_to_copy_files_to_NFS_Server_over_SSH" src="https://user-images.githubusercontent.com/23315232/125184012-75e28400-e212-11eb-8b1d-fcdacbce058d.png">

- Next configure another post-build action for the tooling job to send the build artifacts over SSH 

<img width="582" alt="configuring_post_build_actions_to_Send_build_Artifacts_over_ssh_to_nfs_Server" src="https://user-images.githubusercontent.com/23315232/125184204-bd1d4480-e213-11eb-83f9-6067d433a52f.png">

-In the image above, the post-build action is configured to send all the artifact files and directories to the NFS server using the wild card ``` ** ```

- Make some changes in the Readme.md file for the github repo and commit the changes. Github webhook will trigger the build action in Jenkins and the Jenkins server will 
create artifacts and copy the files over to the NFS server configured

<img width="572" alt="console_output_after_successful_build_files_copied_to_nfs_Server_by_jenkins" src="https://user-images.githubusercontent.com/23315232/125184254-3026bb00-e214-11eb-809e-190e15d9c28a.png">

- Note:
Troubleshooting  permission denied error:

<img width="600" alt="error_encountered_on_jenkins_when_building" src="https://user-images.githubusercontent.com/23315232/125184419-81837a00-e215-11eb-95c1-fe7ce098c838.png">

Configure the ```/mnt/apps``` directory on the NFS server with right permissions the ownership that allows the Jenkins server to copy files to the NFS server

```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

# Ansible configuration management

## Task: 
- Install and configure Ansible client to act as a Bastion host/Jump server
- Create a simple Ansible playbook to automate server configuration
         

## Install and configure Ansible on EC2 instance
- The same EC2 instance used for jenkins in project 9 will be used as Ansible server in this project.
- Create Github repository named `ansble-config-mgt`
- Install Ansible
```
sudo apt update

sudo apt install ansible
```
-Ansible version installed :

<img width="660" alt="ansible_version_installed" src="https://user-images.githubusercontent.com/23315232/126187904-d25cbfe3-51e7-4437-bcda-0b1ed051c257.png">

- Configure Jenkins build job to save the `ansible-config-mgt` repository content every time the repo changes
   - Create a freestyle project in Jenkins
   - configure github webhook and set the webhook to trigger a build in Jenkins
   
   <img width="889" alt="webhook_for_ansible_repo" src="https://user-images.githubusercontent.com/23315232/126189251-155a4ad2-3ba4-40d6-9d64-3ecb6b0fcc95.png">
   
   <img width="867" alt="setting_up_ansible_freestyle_project_on_jenkins_server_to_connect_to_github_repo" src="https://user-images.githubusercontent.com/23315232/126188863-e6d22533-1a9b-4bf4-8e86-7345b283a453.png">
   
     - configure a post-build action to save all `**`  files i.e archive the build
     - Test setup by making changes to the readMe file in the `ansible-config-mgt` repo and commit the changes. This will trigger a build and archive the build files in the     default location 
  ```/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/```
    
    <img width="529" alt="readme_md_file_archived_from_first_post_build_action" src="https://user-images.githubusercontent.com/23315232/126190569-2c8de10f-aaf3-42b6-884d-754a221fd97e.png">
    
- The architecture after setting up jenkins and Ansible on a bastion server is shown below:

<img width="599" alt="architecture_after_setting_up_ansible_and_jekins" src="https://user-images.githubusercontent.com/23315232/126190784-9c33d8b6-986b-4060-a480-e568aee47072.png">
source: https://darey.io

-Assign an Elastic Ip to the Jenkins server to avoid having to reconfigure github webhook everytime you restart the Jenkins server.

<img width="763" alt="assign_elastic_ip_address_to_ansible_server" src="https://user-images.githubusercontent.com/23315232/126191232-3165aa9d-ccb3-48b5-b68b-05ed85969e0a.png">

## Prepare  using VS Code and begin Ansible development
 - setup VSCode to connect to github repo
 - Create a new branch to be used to create new features named ```feature/prj11```
 - Checkout the ```feature/prj11``` branch and create new directories ```inventory``` and ```playbooks``` to be used to store the inventory and playbook files respectively
 - Create a new file named ```common.yml``` in the playbook directory.
 - In the inventory directory create new files `dev.yml`, `staging.yml`, `uat.yml` and `prod.yml` for different environments (development, staging, testing and production environments)
 
 ## setup Ansible inventory
 - The infastructure for the development environment is added to the `inventory/dev.yml` file.
 
 <img width="478" alt="vscode_inventory_file" src="https://user-images.githubusercontent.com/23315232/126194525-cf0445c7-b21a-4957-9cad-74b258676b64.png">

 -  To allow ansible SSH into the target servers. Copy the Private key file for the servers to the Ansible server and ensure the permissions on the file is set to `chmod 400 key.pem` . Also import the key into `ssh-agent`
 ```
 eval `ssh-agent -s`
 ssh-add <path-to-private-key>
 ```
## Create a common playbook
- Update the `playbooks/common.yml` file with the following code which installs wireshark utility on the web, nfs, db and lb servers
```
---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
  - name: ensure wireshark is at the latest version
    yum:
      name: wireshark
      state: latest

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
  - name: ensure wireshark is at the latest version
    apt:
      name: wireshark
      state: latest
```

## Update Git with the latest code
- Commit the changes to the feature brancha and create a pull request (PR) for the branch to be merged into the master branch

<img width="894" alt="creating_pull_request_to_merge_feature_into_master_branch" src="https://user-images.githubusercontent.com/23315232/126197663-ec819881-b8ff-4b33-a1ce-9ca250b99ab0.png">

- Once mergeed into master branch, the github webhook triggers a build in Jenkins and archives the build files

<img width="746" alt="jenkins_build_after_merging_feature_branch_into_master_branch" src="https://user-images.githubusercontent.com/23315232/126198464-3b6e96b2-826b-4478-bef0-0710ae1368ec.png">

<img width="650" alt="post_build_action_jenkins_Archives_playbook_and_inventory_in_jenkins_Server" src="https://user-images.githubusercontent.com/23315232/126198496-9d62ebe6-096e-42c4-b74c-cd6241f92768.png">

## Run first Ansible test
- Execute the common.yml playbook 
```
ansible-playbook -i /var/lib/jenkins/jobs/ansible/builds/<build-number>/archive/inventory/dev.yml /var/lib/jenkins/jobs/ansible/builds/<build-number>/archive/playbooks/common.yml
```

<img width="670" alt="result_of_running_ansible_playbook" src="https://user-images.githubusercontent.com/23315232/126201930-b3c14854-42b8-41bf-ae52-7a76f7d3cc38.png">

- Wireshark utility is  now installed on the target servers

Wireshark version installed on redhat hosts:

<img width="350" alt="wireshark_version_installed_on_ec2-hosts" src="https://user-images.githubusercontent.com/23315232/126201987-9a253c7a-ef64-4587-bab6-d3f2cf5f5106.png">

Wireshark vesion installed on ubuntu hosts:

<img width="329" alt="wireshark_version_installed_on_ubuntu_hosts" src="https://user-images.githubusercontent.com/23315232/126202008-1a48c0f0-c592-499b-9d23-ec20a6b15e23.png">

```
```

## Ansible Refactoring & Static Assignments (Imports and Roles)

This project builds upon the previous project by making improvements of the code and working with the `ansible-config-mgt` repository. The tasks in this project include refactoring the code, creating assignments and learning to use the imports functionality. Imports allow to effectively re-use previously created playbooks in a new playbook - it allows to organize tasks and reuse them when needed.

For better understanding or Ansible artifacts re-use, refer to [ansible docs](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse.html)

- Task:  improve the Ansible code in `ansible-config-mgt` repository developed from project 11 by reorganizing things around a little bit, but the overall state of the infrastructure remains the same.

## Step 1 : Jenkins job enhancement
- Improve the Jenkins job for the ansible config repo to archive artifact in a custom directory named `ansible-config-artifact` instead of creating a new directory for
 each new build in the default directory
 - create a new directory in the jenkins-ansible server named `ansible-config-artifact`
 
   ```sudo mkdir /home/ubuntu/ansible-config-artifact```
 
 - change the permission on the directory to allow jenkins copy archive files to the directory
   ```sudo chmod -R 0777 /home/ubuntu/ansible-config-artifact```

 - Install the `copy-artifact` plugin on the Jenkins-server
 
 <img width="818" alt="install_copy_Artifact_plugin" src="https://user-images.githubusercontent.com/23315232/126717856-85e9ff95-db46-49ad-aaca-416156c78af0.png">

- Create a new freestyle project in jenkins named `save-artifact`. This job/project will be triggered  as a downstream job when the  ansible project is built. The `save-artifact`
  job is configured as shown below to discard old builds and keep only the latest two builds
  
  
  <img width="667" alt="save_Artifact_job_configured_to_discard_old_builds_and_keep_latest_two_builds" src="https://user-images.githubusercontent.com/23315232/126718882-e2daa77f-2de4-4400-b9b1-a7f3029a3597.png">


  <img width="693" alt="save_Artifacts_job_to_trigger_build_after_ansible_project_is_bui;t" src="https://user-images.githubusercontent.com/23315232/126717915-15691a8e-9cae-4683-a6b5-45b0d32a49d1.png">
  
  
  - Configure the `save-artifact` job to save artifacts into /home/ubuntu/ansible-config-artifact directory.
  
  
  <img width="675" alt="configure_build_Step_in_save_Artifacts_job_to_copy_artifacts_from_upstream_job_to_custom_folder" src="https://user-images.githubusercontent.com/23315232/126718979-39ea2b79-ed70-4bbf-9b3c-4b7e1851a64d.png">
  
  
  - Testing the set up above by making changes to the ReadMe.md file. Both the `ansible` job  and the `save-artifact` jobs are built.
  
  
  <img width="415" alt="save-artifact-project-dashboard" src="https://user-images.githubusercontent.com/23315232/126721813-cad2ff00-f285-4ea5-a91e-71d859134fae.png">
  
  ## Step 2 - Refactor ansible code by importing other playbooks into `site.yml`
  - In Project 11 we wrote all tasks in a single playbook common.yml, now it is pretty simple set of instructions for only 2 types of OS, 
    but imagine you have many more tasks and you need to apply this playbook to other servers with different requirements, the single playbook will quickly 
    become complicated and difficult to manage. 
    However, breaking tasks up into different files is an excellent way to organize complex sets of tasks and reuse them
  - Within playbooks folder, create a new file and name it `site.yml` . This file will now be considered as an entry point into the entire infrastructure    
    configuration. Other playbooks will be included here as a reference. 
    In other words, site.yml will become a parent to all other playbooks that will be developed. Including common.yml that you created previously.
  - Create a new folder in root of the repository and name it `static-assignments`. The static-assignments folder is where all other children playbooks will be stored
  - Move common.yml file into the newly created static-assignments folder.
  - Inside site.yml file, import common.yml playbook.  
  
   
  ```
    ----
    - hosts: all
    - import_playbook: ../static-assignments/common.yml
   ```
<img width="613" alt="importing_common_playbook_into_site yml_parent_playbook" src="https://user-images.githubusercontent.com/23315232/126723938-7ba3c57e-09b8-46cc-b322-41ef28b94f96.png">


- Create another playbook under the `static-assignments` directory and name it `common-del` and configure the playbook to delete wireeshark utility from the servers

```
---
- name: update web, and nfs servers
  hosts: webservers, nfs
  remote_user: ec2_user
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    yum:
      name: wireshark
      state: removed  

- name: update LB Server
  hosts: lb, db
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    apt:
      name: wireshark-qt
      state: absent
      autoremove: yes
      purge: yes
      autoclean: yes
```
- update `site.yml` with  `import_playbook: ../static-assignments/common-del.yml`  and run it against the `dev` servers
```
sudo ansible-playbook -i /home/ubuntu/ansible-config-artifact/inventory/dev.yml /home/ubuntu/ansible-config-artifact/playbooks/site.yaml
```
- The wireshark utility is removed from all the targert servers

<img width="403" alt="wireshark_deleted_verified" src="https://user-images.githubusercontent.com/23315232/126724861-3e3e3638-c329-4c4b-b1aa-7f354883dcb6.png">

## Step 3 - Configure UAT Webservers with a role ‘Webserver’
- configure 2 new Web Servers as uat. We could write tasks to configure Web Servers in the same playbook, but it would be too messy, instead, we will use a dedicated     role to make our configuration reusable.
- Launch 2 fresh EC2 instances using RHEL 8 image, we will use them as our uat servers, so give them names accordingly - Web1-UAT and Web2-UAT

<img width="653" alt="uat-servers-snapshot" src="https://user-images.githubusercontent.com/23315232/126725877-298a4338-4876-4bf6-b260-ae83a8d32d51.png">

- To create a role, you must create a directory called `roles/`, relative to the playbook file

- Since the codes are managed on Github, it is recommended practice to create the roles directory manually instead of using ansible-galaxy utility in the jenkins-ansible server
- The directory structure for the roles directory is shown below:

<img width="271" alt="directory_Structure_created_for_ansible_roles" src="https://user-images.githubusercontent.com/23315232/126726151-5b32711f-c0c6-4c36-9f3b-923a07b9ade1.png">
 
- Update the `inventory ansible-config-mgt/inventory/uat.yml` file with IP addresses of the 2 UAT Web servers
 ```
 [uat-webservers]
<Web1-UAT-Server-Private-IP-Address> ansible_ssh_user='ec2-user' ansible_ssh_private_key_file=<path-to-.pem-private-key>
<Web2-UAT-Server-Private-IP-Address> ansible_ssh_user='ec2-user' ansible_ssh_private_key_file=<path-to-.pem-private-key>
 ```
 - In /etc/ansible/ansible.cfg file uncomment roles_path string and provide a full path to your roles directory roles_path = /home/ubuntu/ansible-config-mgt/roles, so Ansible could know where to find configured roles.
 
 <img width="480" alt="etc-ansible-cfg file" src="https://user-images.githubusercontent.com/23315232/126727777-5f374b60-5c24-4458-98cf-e6e88fada8ab.png">
 
 - Add some logic to the webserver role. Go into tasks directory, and within the main.yml file, start writing configuration tasks to do the following:
    - Install and configure Apache (httpd service)
    - Clone Tooling website from GitHub https://github.com/<your-name>/tooling.git.
    - Ensure the tooling website code is deployed to /var/www/html on each of 2 UAT Web servers.
    - Make sure httpd service is started
 
```
 ---
  - name: install apache
   become: yes
   become_user: root
   ansible.builtin.yum:
     name: httpd
     state: present

 - name: install git
   become: yes
   become_user: root
   ansible.builtin.yum:
     name: git
     state: present

 - name: clone a repo
   become: yes
   become_user: root
   ansible.builtin.git:
      repo: https://github.com/jtoguntoye/tooling.git
      dest: /var/www/html
      force: yes

 - name: copy html content one level up
   become: yes
   become_user: root
   command: cp -r /var/www/html/html /var/www/

 - name: start service httpd, if not started
   become: yes
   become_user: root
   ansible.builtin.service:
     name: httpd
     state: started

 - name: recursively remove /var/www/html/html/ directory
   become: yes
   become_user: root
   ansible.builtin.file:
     path: /var/www/html/html
     state: absent
```
 ### Note:
  - The plays in the above playbook must be run as root user to install components, copy folders and start services in the target hosts
 
 ## Step 4 - Reference ‘Webserver’ role
 - Within the `static-assignments` folder, create a new assignment for uat-webservers `uat-webservers.yml`. This is where you will reference the role.
 ```
 ---
 - hosts: uat-webservers
   roles:
     - webserver
 ```
 - Next,  refer to the `uat-webservers.yml` role inside site.yml (which is the entry point for the playbooks)
 ```
 ---
- hosts: all
- import_playbook: ../static-assignments/common.yml

- hosts: uat-webservers
- import_playbook: ../static-assignments/uat-webservers.yml
 ```
## Step 5 - Commit & Test
 - Commit your changes, create a Pull Request and merge them to master branch. Webhook then triggers two consequent Jenkins jobs, and copies all the files to the Jenkins-Ansible server into `/home/ubuntu/ansible-config-artifact/` directory.
 
 <img width="382" alt="artifacts_copied_to_custom_folder_triggered_by_an_upstream_job_in_jenkins" src="https://user-images.githubusercontent.com/23315232/126728932-1ed898f8-b6af-4444-87e1-e41228b28ee1.png">
 
 - run the playbook against the uat inventory
 ```
 sudo ansible-playbook -i /home/ubuntu/ansible-config-artifact/inventory/uat.yml /home/ubuntu/ansible-config-artifact/playbooks/site.yaml
 ```
 
 result:
 
 <img width="630" alt="ansible role successfully installed apache git and cloned repo on uat webservers" src="https://user-images.githubusercontent.com/23315232/126728986-c4303807-4867-4e5f-bb05-7b86b210bcb0.png">

 - Accessing the UAT servers with their public DNS name
 
 <img width="953" alt="reaching_uat_webservers_with_public_dns" src="https://user-images.githubusercontent.com/23315232/126729036-41e5c5fc-4b9c-4d72-a882-caf08b587439.png">
 
 <img width="854" alt="reaching_uat2_webservers_with_public_dns" src="https://user-images.githubusercontent.com/23315232/126736775-362afbca-c166-4d22-b57c-a726aadab313.png">

 
 Final architecture:
 
 
<img width="319" alt="final_Architecture" src="https://user-images.githubusercontent.com/23315232/126729101-137dac62-3f11-4b86-bc72-20af7bffcca0.png">
 

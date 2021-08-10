# Ansible Dynamic Assignments and Community Roles

 This project introduces dynamic assignments by using `include` module.
  - All `import*` statements are pre-processed at the time playbooks are parsed (static assignments).
  - All `include*` statements are processed as they are encountered during the execution of the playbook (Dynamic assignments).
  
  ## Introducing Dynamic assignments into project structure
  - We'll be adding dynamic assignments to the `https://github.com/jtoguntoye/ansible-config-mgt` repository. 
  - Create a new branch named `dynamic-assignments` in the `ansible-config-mgt`repository named `dynamic assignments` and switch to the branch
  
  ```
  git checkout -b dynamic-assignments
  ```
  <img width="567" alt="new_branch_dynamic_Assignment_created" src="https://user-images.githubusercontent.com/23315232/128892221-2f023bf6-c5e2-4d05-8d7c-ddaf5c514090.png">
  
  - create a new folder named `dynamic-assignments` and within this folder a new file named `env-var.yml`. This file will be included later in the `site.yml` playbook 
  - Since we will be using the same Ansible to configure multiple environments, and each of these environments will have certain unique attributes, such as servername,       ip-address etc., we will need a way to set values to variables per specific environment.

 - For this reason, we will now create a folder to keep each environment’s variables file. Therefore, create a new folder env-vars, then for each environment, create new    YAML files which we will use to set variables.
 
 - updated directory structure:
 
 <img width="214" alt="new_ansible_directory_Structure_with_env_vars_and_dynamic_assignments_directory_created" src="https://user-images.githubusercontent.com/23315232/128893596-dea1a690-16f5-469e-b1d6-626da4795c9e.png">
 
 - Next the `env-vars.yml` file will be updated to gather variables from the different server environments that will be configured.
 
 ```
 ---
- name: collate variables form env specific file, if it exists
  hosts: all
  tasks:
    - name: looping through list of all variables
      include_vars: "{{ item }}"
      with_first_found:
       - files:
            - dev.yml
            - stage.yml
            - prod.yml
            - uat. yml
         paths:
           - "{{ playbook_dir }}/../env-vars" 
      tags: 
       - always  
 ```
 - The `env-vars.yml` playbook loops through the variables in the `env-vars` folder using the `include_vars` and `with_first_found` modules. 
 - `{{ playbook_dir }}` is a special variable that helps Ansible to determine the location of the running playbook, and from there navigate to other path on the filesystem 

## Update `site.yml` with dynamic assignments
- Update site.yml file to make use of the dynamic assignment
```
---
- hosts: all
- name: Include dynamic variables 
  tasks:
  import_playbook: ../static-assignments/common.yml 
  include: ../dynamic-assignments/env-vars.yml
  tags:
    - always

-  hosts: webservers
- name: Webserver assignment
  import_playbook: ../static-assignments/webservers.yml
```

## Community Roles
- Next, create a role for MySQL database - it should install the MySQL package, create a database and configure users.
- Download MySQL Ansible role from Ansible [community](https://galaxy.ansible.com/home). In this project, we'll use the mysql role developed by `geerlingguy`. 
- Inside roles directory, create a new MySQL role with ansible-galaxy install `geerlingguy.mysql` and rename the folder to mysql.
-
 

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
- Update site.yml file to make use of the env-vars dynamic assignment
```
---
- hosts: all
- name: Include dynamic variables 
  tasks:
  import_playbook: ../static-assignments/common.yml 
  include: ../dynamic-assignments/env-vars.yml
  tags:
    - always
```

## Community Roles
- Next, create a role for MySQL database - it should install the MySQL package, create a database and configure users.
- Download MySQL Ansible role from Ansible [community](https://galaxy.ansible.com/home). In this project, we'll use the mysql role developed by `geerlingguy`. 
- Inside roles directory, create a new MySQL role with ansible-galaxy install `geerlingguy.mysql` and rename the folder to mysql.

<img width="600" alt="geerlingguy myql_role_installed" src="https://user-images.githubusercontent.com/23315232/128906606-2baff19f-adf5-44fb-bf29-570508d88c2b.png">

- Using the instructions in the README.md file for the role, edit roles configuration to use correct credentials for MySQL required for the tooling website.

<img width="810" alt="configuring_mysql_role_to_use_the_tooling_db" src="https://user-images.githubusercontent.com/23315232/128907380-f6282aaf-cf36-4625-a19e-532dfc6f067e.png">

- Update both the `mysql-server` static-assignment and `site.yml` files to refer the mysql role

```
---
- hosts: db
  roles:
     - mysql
  become: true  
```  

```
--- 
  - name: Import common files
    import_playbook: ../static-assignments/common.yml
    tags:
      - always   
        
  - name: Include dynamic variables
    include: ../dynamic-assignments/env-vars.yml
    tags:
        - always

  - name: database setup
    import_playbook: ../static-assignments/uat-mysqlserver.yml     
  ```
## Load Balancer roles
We want to be able to choose which Load Balancer to use, Nginx or Apache, so we need to have two roles respectively:
- Nginx
- Apache

- Download Nginx and Apache load balancer roles from Ansible community. In this project we'll use roles developed by `gerlingguy`

 <img width="607" alt="geerlingguy nginx_role_installed" src="https://user-images.githubusercontent.com/23315232/128909726-049b4cd6-369a-46de-9d9d-3d1266e17ce0.png">


 <img width="643" alt="geerlingguy apache_role_installed" src="https://user-images.githubusercontent.com/23315232/128909734-b2533dac-e40a-4b04-971a-ee8f4f535273.png">
 
 - Configure the Nginx and Apache LB roles following the instructions in the README.md file for each role
 
 
<img width="566" alt="configuring_apache_loadbalancer_role_for_the_project" src="https://user-images.githubusercontent.com/23315232/128918076-c9b5cfb5-5d7b-4c40-847d-6aa25f921003.png">


<img width="588" alt="configuring_nginx_role_to_ur_project" src="https://user-images.githubusercontent.com/23315232/128918121-d940da7e-e69f-4240-8ca8-e810fbc0eb01.png">

 - Update both  the `load-balancer`static-assignment and site.yml files to refer the roles
 
 
 ```
 - hosts: lb
  roles:
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }
 ```
 
 ```
 --- 
  - name: Import common files
    import_playbook: ../static-assignments/common.yml
    tags:
      - always   
        
  - name: Include dynamic variables
    include: ../dynamic-assignments/env-vars.yml
    tags:
        - always

  - name: database setup
    import_playbook: ../static-assignments/uat-mysqlserver.yml      

  - name: webserver assignment
    import_playbook: ../static-assignments/uat-webservers.yml

  
  - name: loadbalancer assignments
    import_playbook: ../static-assignments/loadbalancers.yml
    when: load_balancer_is_required 
 ```

- Now we can make use of env-vars\uat.yml file to define which loadbalancer to use in UAT environment by setting respective environmental variable to true

```
enable_nginx_lb: true
load_balancer_is_required: true
```
- The same must work with apache LB, so you can switch it by setting respective environmental variable to true and other to false.

- Run the `site.yml` playbook using the uat servers in the `uat` inventory file 
```
ansible-playbook -i inventory/uat playbook/site.yml
```


Output:

```
ubuntu@ip-172-31-10-74://home/ubuntu/ansible-config-mgt$ ansible-playbook -i inventory/uat playbooks/site.yml 
[DEPRECATION WARNING]: 'include' for playbook includes. You should use 'import_playbook' instead. This feature 
will be removed in version 2.12. Deprecation warnings can be disabled by setting deprecation_warnings=False in 
ansible.cfg.

PLAY [update web, nfs and db servers] **************************************************************************

TASK [Gathering Facts] *****************************************************************************************
ok: [172.31.13.108]
ok: [172.31.45.111]
ok: [172.31.15.135]
ok: [172.31.0.84]

TASK [ensure wireshark is at the latest version] ***************************************************************
ok: [172.31.0.84]
ok: [172.31.13.108]
ok: [172.31.45.111]
ok: [172.31.15.135]

PLAY [update LB server] ****************************************************************************************

TASK [Gathering Facts] *****************************************************************************************
ok: [172.31.27.116]

TASK [ensure wireshark is at the latest version] ***************************************************************
ok: [172.31.27.116]

PLAY [collate variables form env specific file, if it exists] **************************************************

TASK [Gathering Facts] *****************************************************************************************
ok: [172.31.45.111]
ok: [172.31.13.108]
ok: [172.31.0.84]
ok: [172.31.15.135]
ok: [172.31.27.116]

TASK [looping through list of all variables] *******************************************************************
ok: [172.31.45.111] => (item=/home/ubuntu/ansible-config-mgt/env-vars/dev.yml)
ok: [172.31.0.84] => (item=/home/ubuntu/ansible-config-mgt/env-vars/dev.yml)
ok: [172.31.13.108] => (item=/home/ubuntu/ansible-config-mgt/env-vars/dev.yml)
ok: [172.31.15.135] => (item=/home/ubuntu/ansible-config-mgt/env-vars/dev.yml)
ok: [172.31.27.116] => (item=/home/ubuntu/ansible-config-mgt/env-vars/dev.yml)

PLAY [db] ******************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************
ok: [172.31.0.84]

TASK [mysql : include_tasks] ***********************************************************************************
included: /home/ubuntu/ansible-config-mgt/roles/mysql/tasks/variables.yml for 172.31.0.84

TASK [mysql : Include OS-specific variables.] ******************************************************************
ok: [172.31.0.84] => (item=/home/ubuntu/ansible-config-mgt/roles/mysql/vars/Debian.yml)

TASK [mysql : Define mysql_packages.] **************************************************************************
ok: [172.31.0.84]

TASK [mysql : Define mysql_daemon.] ****************************************************************************
ok: [172.31.0.84]

TASK [mysql : Define mysql_slow_query_log_file.] ***************************************************************
ok: [172.31.0.84]

TASK [mysql : Define mysql_log_error.] *************************************************************************
ok: [172.31.0.84]

TASK [mysql : Define mysql_syslog_tag.] ************************************************************************
ok: [172.31.0.84]

TASK [mysql : Define mysql_pid_file.] **************************************************************************
ok: [172.31.0.84]

TASK [mysql : Define mysql_config_file.] ***********************************************************************
ok: [172.31.0.84]

TASK [mysql : Define mysql_config_include_dir.] ****************************************************************
ok: [172.31.0.84]

TASK [mysql : Define mysql_socket.] ****************************************************************************
ok: [172.31.0.84]

TASK [mysql : Define mysql_supports_innodb_large_prefix.] ******************************************************
ok: [172.31.0.84]

TASK [mysql : include_tasks] ***********************************************************************************
skipping: [172.31.0.84]

TASK [mysql : include_tasks] ***********************************************************************************
included: /home/ubuntu/ansible-config-mgt/roles/mysql/tasks/setup-Debian.yml for 172.31.0.84

TASK [mysql : Check if MySQL is already installed.] ************************************************************
ok: [172.31.0.84]

TASK [mysql : Update apt cache if MySQL is not yet installed.] *************************************************
skipping: [172.31.0.84]

TASK [mysql : Ensure MySQL Python libraries are installed.] ****************************************************
ok: [172.31.0.84]

TASK [mysql : Ensure MySQL packages are installed.] ************************************************************
ok: [172.31.0.84]

TASK [mysql : Ensure MySQL is stopped after initial install.] **************************************************
skipping: [172.31.0.84]

TASK [mysql : Delete innodb log files created by apt package after initial install.] ***************************
skipping: [172.31.0.84] => (item=ib_logfile0) 
skipping: [172.31.0.84] => (item=ib_logfile1) 

TASK [mysql : include_tasks] ***********************************************************************************
skipping: [172.31.0.84]

TASK [mysql : Check if MySQL packages were installed.] *********************************************************
ok: [172.31.0.84]

TASK [mysql : include_tasks] ***********************************************************************************
included: /home/ubuntu/ansible-config-mgt/roles/mysql/tasks/configure.yml for 172.31.0.84

TASK [mysql : Get MySQL version.] ******************************************************************************
ok: [172.31.0.84]

TASK [mysql : Copy my.cnf global MySQL configuration.] *********************************************************
ok: [172.31.0.84]

TASK [mysql : Verify mysql include directory exists.] **********************************************************
skipping: [172.31.0.84]

TASK [mysql : Copy my.cnf override files into include directory.] **********************************************

TASK [mysql : Create slow query log file (if configured).] *****************************************************
skipping: [172.31.0.84]

TASK [mysql : Create datadir if it does not exist] *************************************************************
ok: [172.31.0.84]

TASK [mysql : Set ownership on slow query log file (if configured).] *******************************************
skipping: [172.31.0.84]

TASK [mysql : Create error log file (if configured).] **********************************************************
skipping: [172.31.0.84]

TASK [mysql : Set ownership on error log file (if configured).] ************************************************
skipping: [172.31.0.84]

TASK [mysql : Ensure MySQL is started and enabled on boot.] ****************************************************
ok: [172.31.0.84]

TASK [mysql : include_tasks] ***********************************************************************************
included: /home/ubuntu/ansible-config-mgt/roles/mysql/tasks/secure-installation.yml for 172.31.0.84

TASK [mysql : Ensure default user is present.] *****************************************************************
skipping: [172.31.0.84]

TASK [mysql : Copy user-my.cnf file with password credentials.] ************************************************
skipping: [172.31.0.84]

TASK [mysql : Disallow root login remotely] ********************************************************************
ok: [172.31.0.84] => (item=DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1'))

TASK [mysql : Get list of hosts for the root user.] ************************************************************
skipping: [172.31.0.84]

TASK [mysql : Update MySQL root password for localhost root account (5.7.x).] **********************************

TASK [mysql : Update MySQL root password for localhost root account (< 5.7.x).] ********************************

TASK [mysql : Copy .my.cnf file with root password credentials.] ***********************************************
skipping: [172.31.0.84]

TASK [mysql : Get list of hosts for the anonymous user.] *******************************************************
ok: [172.31.0.84]

TASK [mysql : Remove anonymous MySQL users.] *******************************************************************

TASK [mysql : Remove MySQL test database.] *********************************************************************
ok: [172.31.0.84]

TASK [mysql : include_tasks] ***********************************************************************************
included: /home/ubuntu/ansible-config-mgt/roles/mysql/tasks/databases.yml for 172.31.0.84

TASK [mysql : Ensure MySQL databases are present.] *************************************************************
ok: [172.31.0.84] => (item={'name': 'tooling', 'collation': 'utf8_general_ci', 'encoding': 'utf8', 'replicate': 1})

TASK [mysql : include_tasks] ***********************************************************************************
included: /home/ubuntu/ansible-config-mgt/roles/mysql/tasks/users.yml for 172.31.0.84

TASK [mysql : Ensure MySQL users are present.] *****************************************************************
changed: [172.31.0.84] => (item=None)
changed: [172.31.0.84]

TASK [mysql : include_tasks] ***********************************************************************************
included: /home/ubuntu/ansible-config-mgt/roles/mysql/tasks/replication.yml for 172.31.0.84

TASK [mysql : Ensure replication user exists on master.] *******************************************************
skipping: [172.31.0.84]

TASK [mysql : Check slave replication status.] *****************************************************************
skipping: [172.31.0.84]

TASK [mysql : Check master replication status.] ****************************************************************
skipping: [172.31.0.84]

TASK [mysql : Configure replication on the slave.] *************************************************************
skipping: [172.31.0.84]

TASK [mysql : Start replication.] ******************************************************************************
skipping: [172.31.0.84]

PLAY [uat] *****************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************
ok: [172.31.13.108]
ok: [172.31.15.135]

TASK [webserver : install apache] ******************************************************************************
ok: [172.31.13.108]
ok: [172.31.15.135]

TASK [webserver : install git] *********************************************************************************
ok: [172.31.13.108]
ok: [172.31.15.135]

TASK [webserver : clone a repo] ********************************************************************************
changed: [172.31.13.108]
changed: [172.31.15.135]

TASK [webserver : copy html content one level up] **************************************************************
changed: [172.31.15.135]
changed: [172.31.13.108]

TASK [webserver : start service httpd, if not started] *********************************************************
ok: [172.31.13.108]
ok: [172.31.15.135]

TASK [webserver : enable service httpd, if not enabled] ********************************************************
ok: [172.31.13.108]
ok: [172.31.15.135]

TASK [webserver : recursively remove /var/www/html/html/ directory] ********************************************
changed: [172.31.13.108]
changed: [172.31.15.135]

PLAY [lb] ******************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************
skipping: [172.31.27.116]

TASK [nginx : Include OS-specific variables.] ******************************************************************
skipping: [172.31.27.116]

TASK [nginx : Define nginx_user.] ******************************************************************************
skipping: [172.31.27.116]

TASK [nginx : include_tasks] ***********************************************************************************
skipping: [172.31.27.116]

TASK [nginx : include_tasks] ***********************************************************************************
skipping: [172.31.27.116]

TASK [nginx : include_tasks] ***********************************************************************************
skipping: [172.31.27.116]

TASK [nginx : include_tasks] ***********************************************************************************
skipping: [172.31.27.116]

TASK [nginx : include_tasks] ***********************************************************************************
skipping: [172.31.27.116]

TASK [nginx : include_tasks] ***********************************************************************************
skipping: [172.31.27.116]

TASK [nginx : Remove default nginx vhost config file (if configured).] *****************************************
skipping: [172.31.27.116]

TASK [nginx : Ensure nginx_vhost_path exists.] *****************************************************************
skipping: [172.31.27.116]

TASK [nginx : Add managed vhost config files.] *****************************************************************

TASK [nginx : Remove managed vhost config files.] **************************************************************

TASK [nginx : Remove legacy vhosts.conf file.] *****************************************************************
skipping: [172.31.27.116]

TASK [nginx : set webservers host name in /etc/hosts] **********************************************************
skipping: [172.31.27.116] => (item={'name': 'web1', 'ip': '172.31.13.108'}) 
skipping: [172.31.27.116] => (item={'name': 'web2', 'ip': '172.31.15.135'}) 

TASK [nginx : Copy nginx configuration in place.] **************************************************************
skipping: [172.31.27.116]

TASK [nginx : Ensure nginx service is running as configured.] **************************************************
skipping: [172.31.27.116]

TASK [apache : Include OS-specific variables.] *****************************************************************
skipping: [172.31.27.116]

TASK [apache : Include variables for Amazon Linux.] ************************************************************
skipping: [172.31.27.116]

TASK [apache : Define apache_packages.] ************************************************************************
skipping: [172.31.27.116]

TASK [apache : include_tasks] **********************************************************************************
skipping: [172.31.27.116]

TASK [apache : Get installed version of Apache.] ***************************************************************
skipping: [172.31.27.116]

TASK [apache : Create apache_version variable.] ****************************************************************
skipping: [172.31.27.116]

TASK [apache : Include Apache 2.2 variables.] ******************************************************************
skipping: [172.31.27.116]

TASK [apache : Include Apache 2.4 variables.] ******************************************************************
skipping: [172.31.27.116]

TASK [apache : Configure Apache.] ******************************************************************************
skipping: [172.31.27.116]

TASK [apache : Ensure Apache has selected state and enabled on boot.] ******************************************
skipping: [172.31.27.116]

PLAY RECAP *****************************************************************************************************
172.31.0.84                : ok=36   changed=1    unreachable=0    failed=0    skipped=23   rescued=0    ignored=0   
172.31.13.108              : ok=12   changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
172.31.15.135              : ok=12   changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
172.31.27.116              : ok=4    changed=0    unreachable=0    failed=0    skipped=0   rescued=0    ignored=0   
172.31.45.111              : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

ubuntu@ip-172-31-10-74://home/ubuntu/ansible-config-mgt$ 
```

Troubleshooting blockers:
- While executing the project, the playbook skipped the load balancer role execution. I'm still working on this. To learn more about configuring LB roles to resolve the blocker.

<img width="815" alt="running_playbook" src="https://user-images.githubusercontent.com/23315232/128922041-321294c0-b2cd-4fd5-94ba-a4e5f10a0815.png">

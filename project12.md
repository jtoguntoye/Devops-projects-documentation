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
  - Within playbooks folder, create a new file and name it `site.yml` . This file will now be considered as an entry point into the entire infrastructure configuration. Other playbooks will be included here as a reference. 
    In other words, site.yml will become a parent to all other playbooks that will be developed. Including common.yml that you created previously.
  - Create a new folder in root of the repository and name it `static-assignments`. The static-assignments folder is where all other children playbooks will be stored
  - Move common.yml file into the newly created static-assignments folder.
  - Inside site.yml file, import common.yml playbook.  
  
   ```
   ---
- hosts: all
- import_playbook: ../static-assignments/common.yml
   ```
<img width="613" alt="importing_common_playbook_into_site yml_parent_playbook" src="https://user-images.githubusercontent.com/23315232/126723938-7ba3c57e-09b8-46cc-b322-41ef28b94f96.png">


 

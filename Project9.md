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




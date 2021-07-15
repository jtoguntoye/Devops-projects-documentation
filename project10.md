# Load Balancer Solution with NGINX and SSL/TLS 

## Task: Configure an Nginx load balancer for Tooling website and ensure connections to the web solution is secured using SSL/TLS certificate

Architecture:

<img width="580" alt="project_infrastrucutre" src="https://user-images.githubusercontent.com/23315232/125762862-881b73e9-d3e0-4e11-a2eb-ac858d46675f.png">
Source: https://Darey.io

## Configure Nginx as a load balancer
- Launch a new Ec2 intance based on Ubuntu server 20.04 LTS named `Nginx LB`
- open port 80 (for http connections) and port 443 (for https connections) on the Nginx LB

<img width="771" alt="security_group_inbound_rules_on_Nginx_lb" src="https://user-images.githubusercontent.com/23315232/125831903-bfcd0726-502a-430c-87f1-40513e7ba18f.png">

- Update `/etc/hosts` file for local DNS settings with web servers' names `web1` and `web2` and their IP addresses

<img width="415" alt="etc-hosts_file_for_nginx_lb" src="https://user-images.githubusercontent.com/23315232/125832140-b6504448-db06-4aa0-8baf-846325bf36e8.png">

- Install and configure Nginx as a load balancer to be used to  point traffic to the web servers 
```
sudo apt update
sudo apt install nginx
```

<img width="482" alt="nginx_server_running_on_lb" src="https://user-images.githubusercontent.com/23315232/125832672-6938b559-6bd9-455b-9af1-36ae7b5911f1.png">


- Configure the Nginx load balancer using web server's name defined in `/etc/hosts` by  editing ```` /etc/nginx/nginx.conf```

<img width="374" alt="configuring_nginx_as_load_balancer" src="https://user-images.githubusercontent.com/23315232/125832739-27b4d9a1-489f-46f8-bc76-389be81b2aa6.png">

- Restart Nginx service and ensure the service is running
```
sudo systemctl restart nginx
sudo systemctl status nginx
```
<img width="482" alt="nginx_server_running_on_lb" src="https://user-images.githubusercontent.com/23315232/125833193-ba52de09-25d2-460a-b215-422c26284a6a.png">

## Register a new domain name and configure secured connection using SSL/TLS
- Register a domain name using domain name registrars like GoDaddy, ClouDNS, or bluehost etc

<img width="472" alt="domain_name_reg" src="https://user-images.githubusercontent.com/23315232/125833656-56b63605-80e4-463a-92e0-bca54c99444a.png">

- Assign an Elastic IP  on AWS to the Nginx load balancer to have a static IP address for the load balancer that does not change on reboot

<img width="465" alt="associate_elastic_ip_to_nginx_lb" src="https://user-images.githubusercontent.com/23315232/125833857-68090737-9f53-46eb-9152-2f0e09bf24fd.png">

- Update an `A record` on the domain name registrar to point to the Nginx lb using the static IP address

<img width="907" alt="A-record-for-domain-name-created'" src="https://user-images.githubusercontent.com/23315232/125834120-742fc5ce-94fb-4984-989e-462eed477485.png">

- Configure Nginx to rcognize your new domain name. Update  `server_name` directive in the ```/etc/nginx/nginx.conf``` file

<img width="627" alt="setting_up_nginx_to_recognize_the_domain_names" src="https://user-images.githubusercontent.com/23315232/125835503-d4338f01-0fb0-4e2b-a248-7b5953a19fb2.png">

- Install Php on the Nginx load balancer to avoid `502 Bad Gateway error` when trying to access the new domain assigned to the LB 
```
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt install php7.4 php7.4-fpm
```
- The domain can now be accessed on the internet, and it loads the tooling website as an unsecured site

- <img width="826" alt="using_domain_name_to_login_unsecured" src="https://user-images.githubusercontent.com/23315232/125836813-be942850-c398-4ba8-8ddb-534ec2d398c9.png">

- Next install Snapd-a software packaging and installing system to be used to install CertBot 
```
sudo apt install snapd
sudo systemctl status snapd
```
- Install CertBot
```
sudo snap install --classic certbot
```
- Request a new SSL/TLS certifcate  for the nginx domain, following the instructions of the certbot
```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```
<img width="581" alt="certbot_successfully_created_TLS_certificate_for_domain" src="https://user-images.githubusercontent.com/23315232/125837715-689ad225-327a-477d-bd92-baa8a055828c.png">

- Configure certbot to renew the certificate at least every 60 days (LetsEncrypt certificate are valid for 90 days)
  - Testing certificate renew command
  ```sudo certbot renew --dry-run```
  - Best practice: Schedule a job that runs the renew command periodically using cronjobs
  ```
  crontab -e
  ```
  -Add the following line to the crontab file to run the renew command twice a day
  ```
  * */12 * * *   root /usr/bin/certbot renew > /dev/null 2>&1
  ```
  
- Secure access to tooling website:
  
 <img width="789" alt="secured_acess_to_tooling_Website" src="https://user-images.githubusercontent.com/23315232/125838162-94c3326e-acf8-40fa-912e-32b9e00cf4ed.png">
  
- Certificate detail:

  <img width="505" alt="certificate_Detail" src="https://user-images.githubusercontent.com/23315232/125838274-3ef20433-432d-4d30-81e5-e13bec41177d.png">

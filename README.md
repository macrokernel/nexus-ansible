# Nexus repository manager + Nginx + Let’s Encrypt SSL in Docker compose  

This Ansible role configures Docker Compose with Sonatype Nexus repository manager, Nginx reverse proxy and Let's Encrypt SSL with auto-renewal.  


## Prerequisites  
This Ansible role requires docker, docker-compose and cron packages to be installed on the system.


## Deployment  
Generate Ansible configs by running the following command
```shell
ansible-playbook ansible/0_generate_configs.yml
```

Edit `ansible/inventory.ini` and specify your Nexus server address and username, for example:
```ini
[nexus]
nexus ansible_host=ubuntu@nexus.example.org
```

Edit `ansible/group_vars/all.yml` and specify your Nexus server DNS name and other parameters, for example:
```yaml
# === Let's Encrypt
certbot_email: admin@example.org
certbot_renew_cron_hour: 3
certbot_renew_cron_minute: 20
certbot_staging: false

# === Nginx
backend_name: nexus
backend_port: 8081
server_name: nexus.example.org
server_alias: repo.example.org repository.example.org

# === Nexus
nexus_api_enable_creation: true
nexus_autoconfiguration: true
```

Run Ansible playbook
```shell
cd ./ansible
ansible-playbook -i inventory.ini 1_install_nexus.yml
```

## Configuration  
Deployment parameters are specified in the `ansible/group_vars/all.yml` configuration file.  

Specify the email address, which will be supplied to Let's Encrypt while requesting SSL certificate, in the `certbot_email` parameter.  

Specify hour and minute of the daily Let's Encrypt certificate renewal cron job in the `certbot_renew_cron_hour` and `certbot_renew_cron_hour` parameters.  

If you wish to test Let's Encrypt SSL certificates issuing, set  `certbot_staging: true` - it helps to avoid hitting Let's Encrypt certificate issuing rate limits while running certbot multiple times.  

The `backend_name` and `backend_port` parameters are used in Nginx configuration file to tell Nginx reverse proxy how to connect to its backend - Nexus.  

The `server_name` and `server_alias` parameters instruct Nginx to forward traffic for the specified hostnames to Nexus. You may specify as many `server_alias` names as you wish, or you may leave it empty or undefined.  

Setting `nexus_api_enable_creation: true` enables Nexus REST API creation features which are disabled in default installation. This in particular allows creation of new repositories using the Nexus REST API.  

The Nexus deployment Ansible role is able to execute a script which configures Nexus using its REST API. If you wish to use this feature, you may edit the `ansible/roles/install_nexus/templates/nexus-autoconfiguration.sh.j2` autoconfiguration script template and set `nexus_api_enable_creation: true` and `nexus_autoconfiguration: true` in `ansible/group_vars/all.yml` before you run the deployment.

Below is an example of the autoconfiguration script which creates a Docker proxy repository:
```shell
#!/bin/sh

curl -u admin:{{ nexus_password }} -k -H 'Content-Type: application/json' 'https://{{ server_name }}/service/rest/v1/script' -d '{"name": "CreateDockerProxy","type": "groovy","content": "repository.createDockerProxy('\''docker-proxy-registry'\'', '\''https://registry-1.docker.io'\'', '\''HUB'\'', null, 5000, null)"}'
curl -X POST -u admin:{{ nexus_password }} -k -H 'Content-Type: text/plain' 'https://{{ server_name }}/service/rest/v1/script/CreateDockerProxy/run'
```

Please note that autoconfiguration can only be performed on the first run of the Nexus deployment role, because Nexus admin password is only known at the first start of Nexus.


## References  
This work is partially based on the following sources.  

[1] How To Secure a Containerized Node.js Application with Nginx, Let's Encrypt, and Docker Compose // https://www.digitalocean.com/community/tutorials/how-to-secure-a-containerized-node-js-application-with-nginx-let-s-encrypt-and-docker-compose  
[2] Setup Nexus repository manager with Nginx reverse proxy and Let’s Encrypt SSL certificate on Docker // https://medium.com/@numb95/setup-nexus-repository-manager-with-nginx-reverse-proxy-and-lets-encrypt-ssl-certificate-on-docker-1c1b05988ce3


## Ansible Role Execution example
```shell
$ ansible-playbook -i inventory.ini 1_install_nexus.yml
[WARNING]: Found both group and host with same name: nexus

PLAY [nexus] **********************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************
ok: [nexus]

TASK [install_nexus : Create /srv/nexus] ******************************************************************************************************************
ok: [nexus]

TASK [install_nexus : Template and copy over the docker-compose file] *************************************************************************************
changed: [nexus]

TASK [install_nexus : Make up server alias parameters string] *********************************************************************************************
skipping: [nexus]

TASK [install_nexus : Update server aliases in certbot command in the docker-compose file] ****************************************************************
skipping: [nexus]

TASK [install_nexus : Enable Lets Encrypt staging mode in certbot command in the docker-compose file] *****************************************************
skipping: [nexus]

TASK [install_nexus : Disable Lets Encrypt staging mode in certbot command in the docker-compose file] ****************************************************
changed: [nexus]

TASK [install_nexus : Create Nexus data directory /srv/nexus/nexus-data] **********************************************************************************
ok: [nexus]

TASK [install_nexus : Create Nginx configuration directory /srv/nexus/nginx-conf] *************************************************************************
ok: [nexus]

TASK [install_nexus : Template and copy over Nginx configuration file "/srv/nexus/nginx.conf"] ************************************************************
ok: [nexus]

TASK [install_nexus : Create Nginx dhparam directory /srv/nexus/dhparam] **********************************************************************************
ok: [nexus]

TASK [install_nexus : Generate dhparam key] ***************************************************************************************************************
ok: [nexus]

TASK [install_nexus : Create Lets Encrypt directories "{{ nexus_docker_compose_location }}/{{ item }}"] ***************************************************
ok: [nexus] => (item=letsencrypt)
ok: [nexus] => (item=web-root)

TASK [install_nexus : Create dummy certificate for "nexus.example.com"] ************************************************************************************
ok: [nexus] => (item=mkdir -p "/srv/nexus/letsencrypt/live/nexus.example.com")
ok: [nexus] => (item=docker-compose run --rm --entrypoint "openssl req -x509 -nodes -newkey rsa:2048 -days 1 -keyout '/etc/letsencrypt/live/nexus.example.com/privkey.pem' -out '/etc/letsencrypt/live/nexus.example.com/fullchain.pem' -subj '/CN=localhost'" certbot)

TASK [install_nexus : Check if Nexus properties file exists] **********************************************************************************************
ok: [nexus]

TASK [install_nexus : Create "/srv/nexus/nexus-data/etc" directory] ***************************************************************************************
ok: [nexus]

TASK [install_nexus : Create "/srv/nexus/nexus-data/etc/nexus.properties" file] ***************************************************************************
changed: [nexus]

TASK [install_nexus : Enable Nexus REST API creation features] ********************************************************************************************
changed: [nexus]

TASK [install_nexus : Create /etc/systemd/system] *********************************************************************************************************
ok: [nexus]

TASK [install_nexus : Template and copy over the systemd service file] ************************************************************************************
ok: [nexus]

TASK [install_nexus : Reload systemd] *********************************************************************************************************************
changed: [nexus]

TASK [install_nexus : Check if Nginx is using dummy certificate] ******************************************************************************************
ok: [nexus]

TASK [install_nexus : Wait for Nginx startup] *************************************************************************************************************
skipping: [nexus]

TASK [install_nexus : Remove dummy certificate for "nexus.example.com"] ************************************************************************************
skipping: [nexus]

TASK [install_nexus : Wait for Certbot to obtain Lets Encrypt certificate for "nexus.example.com"] *********************************************************
skipping: [nexus]

TASK [install_nexus : Restart Nginx container] ************************************************************************************************************
skipping: [nexus]

TASK [install_nexus : Create Lets Encrypt certificate renewal cron job] ***********************************************************************************
ok: [nexus]

TASK [install_nexus : Wait for Nexus to come up] **********************************************************************************************************
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (20 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (19 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (18 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (17 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (16 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (15 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (14 retries left).
FAILED - RETRYING: [nexus]: Wait for Nexus to come up (13 retries left).
ok: [nexus]

TASK [install_nexus : Check if Nexus first-time password file exists] *************************************************************************************
ok: [nexus]

TASK [install_nexus : Look up Nexus admin first-time password] ********************************************************************************************
changed: [nexus]

TASK [install_nexus : set_fact] ***************************************************************************************************************************
ok: [nexus]

TASK [install_nexus : Template and copy over nexus-autoconfiguration.sh] **********************************************************************************
changed: [nexus]

TASK [install_nexus : Execute Nexus autoconfiguration script "nexus-autoconfiguration.sh"] ****************************************************************
changed: [nexus]

TASK [install_nexus : Please change Nexus admin password after first-time login] **************************************************************************
ok: [nexus] => {
    "msg": "Nexus admin password: 2b3d50fd-7bf1-41da-a2fd-0e7e68832617"
}

PLAY RECAP ************************************************************************************************************************************************
nexus                      : ok=27   changed=8    unreachable=0    failed=0    skipped=7    rescued=0    ignored=0
```

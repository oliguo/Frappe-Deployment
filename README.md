# Frappe-Deployment
All in one deployment included ERPNext, CRM, HRMS, Payments, Drive, HelpDesk, Whatsapp via Docker way

# Enviorment
```
OS: Ubuntu 22.04, Installed Docker
```

# Steps
## Step 1, go to default home folder
```
cd ~
```

## Step 2, git clone this repo
```
git clone https://github.com/oliguo/Frappe-Deployment.git frappe

```

## Step 3, start the Traefik(for ssl, domain proxy), PHPMyadmin(for accessing mariadb)
```
sudo docker network create frappe_network
cd ~/frappe/traefik3
sudo docker compose up -d


cd ~/frappe/phpmyadmin-5.2
sudo docker compose up -d
```

Traefik will listen `80`,`443` by default, and ensure your domain pointed to your server IP address.
And you can recheck and ensure there are two folders `letsencrypt` and `logs` created automatically, 
and you also can check the log file `traefik.log` anything wrong.

PHPMyadmin access: `http://your_server_ip_address:8900`
Because we don't have started the frappe sites yet, let's move to next steps.

## Step 4, git clone the frappe docker repo
```
cd ~/frappe
git clone https://github.com/frappe/frappe_docker docker
cd docker
```

## Step 5, copy the files from the folder `frappe_files/*` of this repo to `docker/` of the frappe_docker repo
```
cp ~/frappe/frappe_files/* ~/frappe/docker/
```
There are three files `.env`, `apps.json`,`pwd-custom.yml` to copy to `docker/` folder,
you can adjust the domain, db password on the files `.env` and `pwd-custom.yml`,
by default db root password is `admin`.

## Step 6, building the custom image and start the containers
```
cd ~/frappe/docker/
mkdir -pv db-data logs sites redis-queue-data

export APPS_JSON_BASE64=$(base64 -w 0 apps.json)

sudo docker build \
        --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
        --build-arg=FRAPPE_BRANCH=version-15 \
        --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
        --tag=frappe-all:1.0.1 \
        --file=images/layered/Containerfile .

sudo docker compose -f pwd-custom.yml up -d
```

## Stopped and remove all
```
sudo docker compose -f pwd-custom.yml down
sudo docker volume prune -a
sudo rm -rf db-data logs sites redis-queue-data
```

## There are two containers to be informed 
`frappe_dev-configurator-1` will create some frappe files and settings located in `sites` folder, it will be stopped normally.
`frappe_dev-create-site-1` will check the config files what configurator set, and then go to connect the database, redis , and go to install the applications one by one. It will be stopped normally
It should output the logs correctly like below if you restart it again
```
wait-for-it: waiting 120 seconds for db:3306
wait-for-it: db:3306 is available after 3 seconds
wait-for-it: waiting 120 seconds for redis-cache:6379
wait-for-it: redis-cache:6379 is available after 0 seconds
wait-for-it: waiting 120 seconds for redis-queue:6379
wait-for-it: redis-queue:6379 is available after 0 seconds
sites/common_site_config.json found
Site frontend already exists
App crm already installed
App drive already installed
App helpdesk already installed
App erpnext already installed
App hrms already installed
App raven already installed
App payments already installed
App frappe_whatsapp already installed
```

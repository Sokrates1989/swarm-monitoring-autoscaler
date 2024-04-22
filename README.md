# swarm-monitoring
Bundle of monitoring packages for docker swarm environments


# Prerequisisties
On a multi-master environment you MUST install glusterfs to make prometheus and grafana data be kept accross all nodes.

Otherwise look at portainer installation https://www.portainer.io/blog/monitoring-a-swarm-cluster-with-prometheus-and-grafana

# First Setup

### Subdomain

```text
If you want to use traefik:
Make sure that subdomain exist and points to manager of swarm.
For Example: grafana.felicitas-wisdom.de
```


##### Setup repo at desired location

```bash
# Choose location on server (glusterfs when using multiple nodes is recommended).
mkdir -p /gluster_storage/swarm/administration/monitoring-autoscaler
cd /gluster_storage/swarm/administration/monitoring-autoscaler
git clone https://github.com/Sokrates1989/swarm-monitoring-autoscaler.git .
```

##### Create secret in docker swarm
```bash
# GRAFANA LOGIN.
vi secret.txt  # Then insert password (Make sure the password does not contain any backslashes "\") and save the file.
docker secret create SWARM_MONITORING_GRAFANA_PASSWORD secret.txt 
rm secret.txt

# GRAFANA SMTP PASSWORD.
vi secret.txt  # Then insert password (Make sure the password does not contain any backslashes "\") and save the file.
docker secret create SWARM_MONITORING_GRAFANA_SMTP_PASSWORD secret.txt 
rm secret.txt
```

##### Configuration
```bash
# Copy ".env.template" to ".env".
cp .env.template .env

# Edit .env
vi .env
```


###### chmod

```bash
# Allow to read and write data.
chmod -R 777 grafana_data/
chmod -R 777 prometheus_data/
```

# Deploy

##### Option 1: Requires traefik (RECOMMENDED)
```bash
# Allows you to access api using the url (GRAFANA_URL) provided in .env.
# But requires you to have traefik set up.
docker stack deploy -c <(docker-compose -f config-stack-traefik.yml config) monitoring-autoscaler
```

##### Option 2: Does not require traefik, uses default ports
```bash
# You can only call the api using http://<MANAGER_IP_ADDRESS>:3000/.
# NOT ENCRYPTED / NOT RECOMMENDED / JUST FOR DEBUGGING, TESTING.
docker stack deploy -c <(docker-compose -f config-stack.yml config) monitoring-autoscaler
```

# Usage

##### Option 1: Using subdomain entered in .env / Requires traefik
- Open Browser and enter https://grafana.felicitas-wisdom.de (the url set for GRAFANA_URL in .env)

##### Option 2: Using IP Address and Port / Does not require traefik
- Retrieve IP Address of -> MANAGER_IP_ADDRESS
- Open Browser and enter http://<MANAGER_IP_ADDRESS>:3000/
- In case of Errors: Ensure, that port 3000 is accessible

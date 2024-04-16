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
mkdir -p /gluster_storage/swarm/monitoring/monitoring
cd /gluster_storage/swarm/monitoring/monitoring
git clone https://github.com/Sokrates1989/swarm-monitoring.git .
```

##### Configuration
```bash
# Copy ".env.template" to ".env".
cp .env.template .env

# Edit .env
vi .env
```

# Deploy

##### Option 1: Requires traefik (RECOMMENDED)
```bash
# Allows you to access api using the url (monitoring_URL) provided in .env.
# But requires you to have traefik set up.
docker stack deploy -c <(docker-compose -f docker-compose-traefik.yml config) monitoring
```

##### Option 2: Does not require traefik, uses default ports
```bash
# You can only call the api using http://<MANAGER_IP_ADDRESS>:3000/.
# NOT ENCRYPTED / NOT RECOMMENDED / JUST FOR DEBUGGING, TESTING.
docker stack deploy -c <(docker-compose config) monitoring
```

# Usage

##### Option 1: Using subdomain entered in .env / Requires traefik
- Open Browser and enter https://monitoring.felicitas-wisdom.de (the url set for monitoring_URL in .env)

##### Option 2: Using IP Address and Port / Does not require traefik
- Retrieve IP Address of -> MANAGER_IP_ADDRESS
- Open Browser and enter http://<MANAGER_IP_ADDRESS>:3000/
- In case of Errors: Ensure, that port 3000 is accessible

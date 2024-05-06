# swarm-monitoring-autoscaler
Bundle of monitoring packages for docker swarm environments with autoscaler to scale your swarm services based on cpu or memory consumption


## Table of contents
1. [Included packages](#included-packages)
2. [Prerequisites](#prerequisites)
   - [Swarm Cronjobs](#swarm-cronjobs)
   - [If on a multiple master swarm](#if-on-a-swarm-cluster-with-multiple-masters)
     - [Option1: Glusterfs or any other distributed, arbitrarily scalable file system like Ceph (recommended)](#option1-glusterfs-or-any-other-distributed-arbitrarily-scalable-file-system-like-ceph-recommended)
     - [Option2: Constrain deployment to a specific node](#option2-constrain-deployment-to-a-specific-node)
   - [Optional: Traefik (recommended)](#optional-traefik-recommended)
3. [First Setup](#first-setup)
4. [Deploy](#deploy)
5. [Usage](#usage)
   - [AutoScaler](#autoscaler)
     - [Quick overview of labels](#autoscaler)
     - [Detailed information of labels](#full-explanation)
     - [Check Logs](#logs)
   - [Grafana](#grafana)
     - [View Autoscaler Metrics](#view-autoscaler-metrics)
     - [Dashboards](#dashboards)


# Included Packages
- autoscaler
  - Scale swarm services based on cpu/memory consumption
- grafana 
  - Visualize your swarm's metrics, optimized for autoscaler with various helpful pre-built dashboards
- prometheus 
  - API for metric data (Gets data from cadvisor and node-exporter and provides them for grafana and autoscaler)
- cadvisor
  - Fetches metrics from swarm services/ containers
- node-exporter
  - Monitors the host system


# Prerequisites
## Swarm Cronjobs
To make autoscaler run every 15 seconds (default, can be customized in .env) deploy https://github.com/crazy-max/swarm-cronjob or another way to implement peridical launch of services.

Implementation help can be found at https://github.com/Sokrates1989/swarm-cronjob.git.

More infromation
 - https://crazymax.dev/swarm-cronjob/
 - https://pkg.go.dev/github.com/robfig/cron#hdr-CRON_Expression_Format

## If on a swarm cluster with multiple masters

### Option1: Glusterfs or any other distributed, arbitrarily scalable file system like Ceph (recommended)

!! This is recommended to avoid a single point of failure causing autoscaler, grafana and prometheus to go down !!

On a multi-master environment: Install glusterfs to make autoscaler, prometheus and grafana data be kept accross all nodes.
- https://docs.techdox.nz/glusterfs/#
- https://docs.gluster.org/en/main/Install-Guide/Install/
- https://docs.gluster.org/en/main/


### Option2: Constrain deployment to a specific node

!! Keep in mind, that if that node fails, also prometheus and grafana in the whole cluster will not work !!

Setup deploy constraints to run grafana and prometheus only on one specific node. 

#### Label creation

```bash
# Get nodeID on which you want to deploy grafana and prometheus on.
docker node ls

# Set label on that node.
docker node update --label-add monitoring=true <nodeID_withManyNumbersAndLetters>

# Verify, that label is set correctly.
docker node ls -q | xargs docker node inspect --format '{{ .ID }} [{{ .Description.Hostname }}]: {{ .Spec.Labels }}'

# OPTIONAL/ JUST INFO: In case you want to remove label/ change monitoring node / ...
docker node update --label-rm <label_key> <nodeID_withManyNumbersAndLetters>
docker node update --label-rm monitoring <nodeID_withManyNumbersAndLetters>
```

#### Deploy on monitoring node
Make sure to [setup repo](#setup-repo-at-desired-location) on the node where you just set the monitoring label on

#### Uncomment monitoring label
When you get to [deploy section](#deploy) -> edit config-stack.yml or config-stack-traefik.yml 
```bash
# If you want to use traefik.
vi config-stack-traefik.yml

# If you do not want to use traefik.
vi config-stack.yml
```

look for 
```text
        # - node.labels.monitoring == true
```
and remove "# " so that it look like below. Make sure "-" is in same vertical place as other labels.
```text
        - node.role == manager
        ...
        - node.labels.monitoring == true
```


## Optional: Traefik (recommended)
If you want to access grafana with its own subdomain and via HTTPS. It is recommended, because otherwise you access grafana with a published port and insecurely (HTTP)
- https://traefik.io/traefik/
- https://github.com/traefik/traefik
- https://github.com/Sokrates1989/swarm_traefik

# First Setup

### Subdomain

```text
If you want to use grafana with subdomain in combination with traefik:
Make sure that subdomain exist and points to manager of swarm.
For Example: grafana.felicitas-wisdom.de
```


### Setup repo at desired location
```bash
# Choose location on server (glusterfs when using multiple nodes is recommended).
mkdir -p /gluster_storage/swarm/administration/monitoring-autoscaler
cd /gluster_storage/swarm/administration/monitoring-autoscaler
git clone https://github.com/Sokrates1989/swarm-monitoring-autoscaler.git .
```

### Create secret in docker swarm
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

### Configuration
```bash
# Copy ".env.template" to ".env".
cp .env.template .env

# Edit .env
vi .env
```


#### chmod

```bash
# Allow to read and write data.
chmod -R 777 grafana_data/
chmod -R 777 prometheus_data/
```

# Deploy

#### Option 1: Requires traefik (RECOMMENDED)
```bash
# Allows you to access api using the url (GRAFANA_URL) provided in .env.
# But requires you to have traefik set up.
docker stack deploy -c <(docker-compose -f config-stack-traefik.yml config) monitoring-autoscaler
```

#### Option 2: Does not require traefik, uses default ports
```bash
# You can only access grafana using http://<MANAGER_IP_ADDRESS>:3000/.
# NOT ENCRYPTED / NOT RECOMMENDED / JUST FOR DEBUGGING, TESTING.
docker stack deploy -c <(docker-compose -f config-stack.yml config) monitoring-autoscaler
```

# Usage

## AutoScaler

When deploying another new service, that you want to autoscale, you add labels to the deploy section of that service.

#### Short info
```yaml
version: "3.9"

services:
 your_service:
    image: ...
    ...
    deploy:
      labels: ### PLACE YOUR LABELS HERE ###
        ...

        # Enable autoscaling for your service.
        - "autoscale=true"

        # How many replicas do you want?
        - "autoscale.minimum_replicas=1"
        - "autoscale.maximum_replicas=3"

        # What do you want to base scaling on?

        # CPU?
        - "autoscale.cpu_upscale_threshold=80"
        - "autoscale.cpu_upscale_time_duration=2m"
        - "autoscale.cpu_downscale_threshold=20"
        - "autoscale.cpu_downscale_time_duration=5m"

        # Or Memory? (or even both?) 
        - "autoscale.memory_upscale_threshold=" 
        - "autoscale.memory_downscale_threshold=" 

        # What do you want to do if one metric says scale down and the other says scale up/ otherwise?
        - "autoscale.scaling_conflict_resolution=scale_up"

        # Individual log level for this service.
        - "autoscale.log_level=INFO"

        # For more info read below full explanation.
        ...
    ...
```


#### Full explanation
```yaml
version: "3.9"

services:
 your_service:
    image: ...
    ...
    deploy:
      labels:
        ...

        ### Common and required labels ###

        - "autoscale=true" # REQUIRED. Must be set to true to enable autoscaling
        - "autoscale.minimum_replicas=1" # REQUIRED. The minimum amount of replicas you want to keep at least (integer, min 1)
        - "autoscale.maximum_replicas=3" # REQUIRED. The maximum amount of replicas you want to scale up to (integer)



        ### CPU related labels ###

        - "autoscale.cpu_upscale_threshold=80" # When average service cpu of the last 2 minutes (autoscale.cpu_upscale_time_duration) rises above this level -> scale up (Look at Grafana to retrieve current cpu avg to get a grasp of cpu consumption over time duration)
        - "autoscale.cpu_upscale_time_duration=2m" # The prometheus time duration (https://prometheus.io/docs/prometheus/latest/querying/basics/#time-durations) to base the query of cpu upscaling on. To react more quickly to increases of of cpu load it should be shorter than cpu_downscale_time_duration. Should be a multiplier of 30s.
        - "autoscale.cpu_downscale_threshold=20" # When average service cpu of the last 5 minutes (autoscale.cpu_downscale_time_duration) sinks below this level -> scale down (Look at Grafana to retrieve current cpu avg to get a grasp of levels)
        - "autoscale.cpu_downscale_time_duration=5m" # The prometheus time duration (https://prometheus.io/docs/prometheus/latest/querying/basics/#time-durations) to base the query of cpu downscaling on. To avoid downscaling too early it should be longer than cpu_upscale_time_duration. Should be a multiplier of 30s.



        ### Memory related labels ###

        - "autoscale.memory_upscale_threshold=" # When memory consumption rises above this level -> scale up (Look at Grafana to retrieve current memory metrics to get a grasp of levels)
        - "autoscale.memory_downscale_threshold=" # When memory consumption of  sinks below this level -> scale down(Look at Grafana to retrieve current memory metrics to get a grasp of levels)



        ### Optional settings ###

        - "autoscale.scaling_conflict_resolution=scale_up" # OPTIONAL. (scale_up | scale_down | keep_replicas | adhere_to_memory | adhere_to_cpu) When metric1 says the service should scale up, but metric2 says "scale down" -> here you choose how you want to handle this szenario (default is scale_up, no need to define label in the default case, but advised )
        - "autoscale.log_level=INFO"  # OPTIONAL. (INFO | VERBOSE | WARNING_AND_ERRORS_ONLY) Service based custom log level. You can fine-tune the log level for each autoscale service. This can be handy for debugging or when you just set up a new service and want to see if everything works fine.
        
        ...
    ...
```

### Logs

#### Hint / Tip
Deploy autoscale services with verbose log level to get more information or if you want to debug a potential error or want to see how the autoscaler actually works. (Remember to change back to INFO or WARNING_AND_ERRORS_ONLY in permanent production)
```yaml
        - "autoscale.log_level=VERBOSE"
```

#### Check Logs

After having setup this stack and deployed a service with autoscale labels set, always check the logs, to ensure autoscaler is working correctly


##### Check logs via cli
```bash
# Current static output of logs.
docker service logs monitoring-autoscaler_autoscaler

# Check for changes in the logs every 2 seconds.
watch docker service logs monitoring-autoscaler_autoscaler
```

##### Check logs using files
```bash
# Go to src dir of log files.
cd /gluster_storage/swarm/administration/monitoring-autoscaler/autoscaler_logs
# Optionally go to dayBased directory.

# View content of directories.
ll # or equivalent: ls -al
ll dayBased/ # or equivalent: ls -al dayBased/

# View content of logfiles.
vi log.txt # Opens file with editor vi.
tail log.txt # Print the last few lines of file to cli.
cat log.txt # Print whole file to cli.
```


## Grafana

### Access using browser 

##### Option 1: Using subdomain entered in .env / Requires traefik
- Open Browser and enter https://GRAFANA_URL (the url you set for GRAFANA_URL in .env)

##### Option 2: Using IP Address and Port / Does not require traefik
- Retrieve IP Address of -> MANAGER_IP_ADDRESS
- Open Browser and enter http://<MANAGER_IP_ADDRESS>:3000/
- In case of Errors: Ensure, that port 3000 is accessible

### Login 
Login using GRAFANA_USER provided in .env and the secret SWARM_MONITORING_GRAFANA_PASSWORD created beforehand

### View Autoscaler Metrics
 - Head over to Dashboards in the left menu and select the Autoscaler Metrics Dashboard. 
 - Here you can see the cpu and memory consumption of your services.
 - Based on that data, you can easily setup the autoscaler threshold values.

### Dashboards and Alerts
#### Dashboards
There are 3 dashboards pre-created for your convenience:
 - Autoscaler Metrics
   - information about your services using data from cadvisor via prometheus
   - quickly view the metrics needed to scale your services for autoscaler
 - Container Metrics
   - information about your containers also using data from cadvisor via prometheus
 - Node Metrics
   - information about your host machines using data from node-exporter via prometheus

#### Alerts from dashboard panels
If you want to be informed, if any metric you see on the dashboard changes above or below a certain value, you can directly create an alert rule from that panel. Click on the context menu of the panel (top right hand corner) and select "New alert rule" (in sub-menu "More...")

### More information about Grafana
 - https://grafana.com/
 - https://grafana.com/docs/grafana/latest/
 - https://grafana.com/docs/grafana/latest/introduction/
 - https://grafana.com/docs/grafana/latest/dashboards/
 - https://grafana.com/docs/grafana/latest/panels-visualizations/
 - https://grafana.com/docs/grafana/latest/alerting/

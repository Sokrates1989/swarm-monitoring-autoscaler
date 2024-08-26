# swarm-monitoring-autoscaler
Bundle of monitoring packages for docker swarm environments with autoscaler to scale your swarm services based on cpu or memory consumption


## Table of contents
1. [Included packages](#included-packages)
2. [Backlog](#backlog)
3. [Prerequisites](#prerequisites)
   - [Swarm Cronjobs](#swarm-cronjobs)
   - [If on a multiple master swarm](#if-on-a-swarm-cluster-with-multiple-masters)
     - [Option1: Glusterfs or any other distributed, arbitrarily scalable file system like Ceph (recommended)](#option1-glusterfs-or-any-other-distributed-arbitrarily-scalable-file-system-like-ceph-recommended)
     - [Option2: Constrain deployment to a specific node](#option2-constrain-deployment-to-a-specific-node)
   - [Optional: Traefik (recommended)](#optional-traefik-recommended)
4. [First Setup](#first-setup)
   - [Optional: Telegram status messages](#optional-telegram-status-messages)
   - [Optional: Autoscaler State Checker](#optional-autoscaler-state-checker)
5. [Deploy](#deploy)
6. [Usage](#usage)
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


# Backlog
- StateChecker Client
- More scaling options
  - network (upstream/ downstream)
  - Filesystem IO (read/ write)
  - Response Time / Latency
  - Request Rate (Throughput)
  - Custom Metrics (whatever prometheus allows to retrieve) ? 
    - queue length in message queues?
    - number of active sessions?
    - database connection pool usage?
  - Database Metrics?
    - query response time?
    - connection pool usage?
    - transaction rates?
- Grafana pre built dashboards based on new scaling metrics
- Go live/ make public
  - Lizenz
  - Chat GPT chat help
    - Archived Chat?
    - How to promote?
  - Donations / Add to README
    - Buy me a coffee
    - Paypal 

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


### Create secrets in docker swarm

All Secrets must be created for the stack to work. If you are not indending to use a secret, you can just create the secret with the text "none".
```bash
### AUTOSCALER ###

# AUTOSCALER TELEGRAM SENDER BOT TOKEN for status messages.
# This is the bot token the bot father gave when setting up status messages via telegram.
# Insert "none", if you do not want to use telegram status messages.
vi secret.txt  # Then insert password (Make sure the password does not contain any backslashes "\") and save the file.
docker secret create SWARM_MONITORING_AUTOSCALER_TELEGRAM_SENDER_BOT_TOKEN secret.txt 
rm secret.txt

# AUTOSCALER EMAIL SENDER PASSWORD for status mails.
# This is the password you use to log in to your email provider to send mails (SMTP).
# Insert "none", if you do not want to use email status messages.
vi secret.txt  # Then insert password (Make sure the password does not contain any backslashes "\") and save the file.
docker secret create SWARM_MONITORING_AUTOSCALER_EMAIL_SENDER_PASSWORD secret.txt 
rm secret.txt


### AUTOSCALER STATECHCKER ###

# AUTOSCALER STATECHECKER SERVER AUTHENTICATION TOKEN.
# This is the server authentication token for the statechecker client to verify authentication.
# Insert "none", if you do not want to use statechecker for autoscaler.
vi secret.txt  # Then insert token and save the file.
docker secret create SWARM_MONITORING_AUTOSCALER_STATECHECKER_SERVER_AUTH_TOKEN secret.txt 
rm secret.txt

# AUTOSCALER STATECHECKER TOOL TOKEN.
# This is the token for the server to verify this tools authentication.
# Insert "none", if you do not want to use statechecker for autoscaler.
vi secret.txt  # Then insert token and save the file.
docker secret create SWARM_MONITORING_AUTOSCALER_STATECHECKER_TOOL_TOKEN secret.txt 
rm secret.txt

# AUTOSCALER STATECHECKER TELEGRAM SENDER BOT TOKEN for status messages.
# This is the bot token the bot father gave when setting up status messages via telegram for statechecker bot.
# Insert "none", if you do not want to use statechecker telegram status messages.
vi secret.txt  # Then insert bot token and save the file.
docker secret create SWARM_MONITORING_AUTOSCALER_STATECHECKER_TELEGRAM_BOT_TOKEN secret.txt 
rm secret.txt


### GRAFANA ###

# GRAFANA LOGIN.
vi secret.txt  # Then insert password (Make sure the password does not contain any backslashes "\") and save the file.
docker secret create SWARM_MONITORING_GRAFANA_PASSWORD secret.txt 
rm secret.txt

# GRAFANA SMTP PASSWORD.
# Insert "none", if you do not want to use grafana email status messages.
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



### Optional: Telegram status messages

#### Create a telegram bot
- Talk to [Botfather](https://t.me/BotFather) https://t.me/BotFather using telegram
- Open the menu and select "/newbot"
- Follow the instructions starting with giving your Bot a name such as "Swarm Autoscale Status Messages" or "autoscale_status_message_bot"
- Obtain and copy the token to a save place. This is your "login-password" authenticating you to send messages via this bot

#### Create chats and chat groups your new bot can talk to
Your new bot cannot simply send messages to anyone who uses telegram. We must allow the bot to talk with [groups](#for-any-telegram-group) or [individual users](#for-the-current-telegram-user).

##### For the current telegram user
- In telegram start a new chat by searching for the bot name of your newly created bot: e.g.: "autoscale_status_message_bot"
- Within the chat press "START" and write any message such as "Hi"
- Go on at [Obtain chat ID of recipient](#obtain-chat-id-of-recipient)


##### For any telegram group
This is handy as you can easily create groups for services, information levels or clusters and then simply add or remove people who should have access to this information. This moves the administration to a much easier to manage location with a corresponding understandable user interface.

- In telegram start a new empty group (e.g.: "Important Autoscaler Messages") DO NOT ADD THE THE BOT IN THE DIALOG DIRECTLY
- Add the bot as a new member () (autoscale_status_message_bot)
- Go on at [Obtain chat ID of recipient](#obtain-chat-id-of-recipient)


##### Obtain chat ID of recipient
- Build a url to retrieve the chat id after sending a message to the bot or the group chat above: [Link to retrieve chat id with telegram bot](#link-to-retrieve-chat-id-with-telegram-bot)
- Open that url in a browser and look for the message you just sent to the bot and retrieve the chat id
  - [For individual users](#example-return-for-indivdual-chat-url)
  - [For groups](#example-return-for-group-chat-url)
- Make a note of the chat ID and add it to the desired recipients of the log level that this user should receive


##### Link to retrieve chat id with telegram bot
```yml
# Replace <YOUR_NEW_BOT_TOKEN_OBTAINED_FROM_BOTFATHER> with the bot token of your bot.
https://api.telegram.org/bot<YOUR_NEW_BOT_TOKEN_OBTAINED_FROM_BOTFATHER>/getUpdates

# It should now look somehting like this.
https://api.telegram.org/bot1234567890:AAA0AAaaa_AAAA0A0aA0a0aAA00a0/getUpdates
```


<details>
<summary>How to find chat id from webpage response</summary>

##### Example return for indivdual [chat url](#link-to-retrieve-chat-id-with-telegram-bot)
```json
// Original return.
{"ok":true,"result":[{"update_id":123456789,
"message":{"message_id":2,"from":{"id":1234567890,"is_bot":false,"first_name":"FIRSTNAME","username":"USERNAME","language_code":"de"},"chat":{"id":1234567890,"first_name":"FIRSTNAME","username":"USERNAME","type":"private"},"date":1234567890,"text":"Hi"}}]}

// Reformated to help you understand what to look out for.
{
  "ok": true,
  "result": [
    {
      "update_id": 123456789,
      "message": 
      {
        "message_id": 2,
        "from": 
        {
          "id": 123456789,
          "is_bot": false,
          "first_name": "FIRSTNAME",
          "username": "USERNAME",
          "language_code": "de"
        },
        "chat": // Chat object containing the id we are looking for
        {
          "id": 123456789, // <- This is the chat id we want
          "first_name": "FIRSTNAME",
          "username": "USERNAME",
          "type": "private"
        },
        "date": 123456789,
        "text": "Hi"
      }
    }
  ]
}
```

##### Example return for group [chat url](#link-to-retrieve-chat-id-with-telegram-bot)
```json
// Original return.
{"ok":true,"result":[{"update_id":123456789,"message":{"message_id":3,"from":{"id":1234567890,"is_bot":false,"first_name":"Patrick","username":"Patrick_IT_Cologne","language_code":"de"},"chat":{"id":-1234567890,"title":"ImportantAutoscalerMessages","type":"group","all_members_are_administrators":true},"date":1234567890,"new_chat_participant":{"id":1234567890,"is_bot":true,"first_name":"SwarmAutoscaleStatusMessages","username":"autoscale_status_message_bot"},"new_chat_member":{"id":1234567890,"is_bot":true,"first_name":"SwarmAutoscaleStatusMessages","username":"autoscale_status_message_bot"},"new_chat_members":[{"id":1234567890,"is_bot":true,"first_name":"SwarmAutoscaleStatusMessages","username":"autoscale_status_message_bot"}]}}]
}

// Reformated to help you understand what to look out for.
{
  "ok": true,
  "result": [
    {
      "update_id": 123456789,
      "message": 
      {
        "message_id": 3,
        "from": 
        {
          "id": 1234567890,
          "is_bot": false,
          "first_name": "Patrick",
          "username": "Patrick_IT_Cologne",
          "language_code": "de"
        },
        "chat": // Chat object containing the id we are looking for
        { 
          "id": -1234567890, // <- This is the chat id we want (group ids often start with - (keep it as part of the id))
          "title": "Important Autoscaler Messages",
          "type": "group",
          "all_members_are_administrators": true
        },
        "date": 1234567890,
        "new_chat_participant": 
        {
          "id": 1234567890,
          "is_bot": true,
          "first_name": "Swarm Autoscale Status Messages",
          "username": "autoscale_status_message_bot"
        },
        "new_chat_member":  // That is why we add the bot afterwards as we can see this message 
        {
          "id": 1234567890,
          "is_bot": true,
          "first_name": "Swarm Autoscale Status Messages",
          "username": "autoscale_status_message_bot"
        },
        "new_chat_members": 
        [
          {
              "id": 1234567890,
              "is_bot": true,
              "first_name": "Swarm Autoscale Status Messages",
              "username": "autoscale_status_message_bot"
          }
        ]
      }
    }
  ]
}
```
</details>


### Optional: Autoscaler state checker
The autoscale-state-checker will check, if the autoscaler is working as expected. It will send messages, in case it does not. After autoscaler completes its tasks, it will ping the state-checker-server, to tell it everything worked as expected.

#### Setup 
Follow instructions of https://github.com/Sokrates1989/docker-stateChecker-server/

#### Enable and configuration adaptions




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
      labels: ### <- PLACE AUTOSCALER LABELS HERE ###
        ...

        ### Common and required labels ###

        # Enable autoscaling for your service.
        - "autoscale=true"

        # How many replicas do you want?
        - "autoscale.minimum_replicas=1"
        - "autoscale.maximum_replicas=3"


        ### What do you want to base scaling on ? ###

        # CPU?
        - "autoscale.cpu_upscale_threshold=80"
        - "autoscale.cpu_upscale_time_duration=2m"
        - "autoscale.cpu_downscale_threshold=20"
        - "autoscale.cpu_downscale_time_duration=5m"

        # Or Memory? (or even both?) 
        - "autoscale.memory_upscale_threshold=" 
        - "autoscale.memory_downscale_threshold=" 

        
        ### Optional settings ###

        # What do you want to do if one metric says scale down and the other says scale up/ otherwise?
        - "autoscale.scaling_conflict_resolution=scale_up"

        # Overwrite log level for this service.
        - "autoscale.log_level=INFO"


        ### Additional status messages for this service ###

        # Who additionally do you want to send status messages via email for this service?
        - "autoscale.additional_email_recipients_important_msgs="
        - "autoscale.additional_email_recipients_information_msgs="
        - "autoscale.additional_email_recipients_verbose_msgs="

        # Who additionally do you want to send what status messages via telegramm for this service?
        - "autoscale.additional_telegram_recipients_important_msgs="
        - "autoscale.additional_telegram_recipients_information_msgs="
        - "autoscale.additional_telegram_recipients_verbose_msgs="

        
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
        - "autoscale.log_level=INFO"  # OPTIONAL. (INFO | VERBOSE | IMPORTANT_ONLY) Service based custom log level. You can fine-tune the log level for each autoscale service. This can be handy for debugging or when you just set up a new service and want to see if everything works fine.


        ### Additional status messages for this service ###

        # Who additionally do you want to send status messages via email for this service?
        # Comma seperated email addresses.
        - "autoscale.additional_email_recipients_important_msgs=mail@example.com,mail2@example.com"
        - "autoscale.additional_email_recipients_information_msgs=mail@example.com"
        - "autoscale.additional_email_recipients_verbose_msgs="

        # Who additionally do you want to send what status messages via telegramm for this service?
        # Comma seperated telegram chatIDs.
        - "autoscale.additional_telegram_recipients_important_msgs="
        - "autoscale.additional_telegram_recipients_information_msgs="
        - "autoscale.additional_telegram_recipients_verbose_msgs="

        ...
    ...
```

### Logs

#### Hint / Tip
Deploy autoscale services with verbose log level to get more information or if you want to debug a potential error or want to see how the autoscaler actually works. (Remember to change back to INFO or IMPORTANT_ONLY in permanent production)
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

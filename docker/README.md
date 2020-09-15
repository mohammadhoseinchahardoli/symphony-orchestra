# Docker
## Installation
## Log Management
### Syslog
#### Background
During one of the projects that I worked on in recent years, I had the task of integrating a centralized logging system
with the applications stack we use (following a microservice architecture). The first idea that came to mind was to
build an ELK (Elasticsearch, Logstash, Kibana) stack to accomplish this task. However, it turned out that this was not
an option due to a limitation on the project’s resources.
As a result, I investigated all the options and ideas that would help me implement this feature without conflicting
with our limitations. I ultimately came up with the following ideas:

- Collect Docker logs directly.
- Using Syslog to collect the logs.

As I already said, the first idea was to build an ELK stack for collecting the logs. However, that was not an option
due to budget and resource limitations for the project.
Another idea was to forward the logs to a managed service like logz.io. This option was also not possible because
production servers have no internet access—plus the same budget limitations.
The third idea was to set up a Logrotate script that performs the following actions to generate the log files:

- Collect the Docker logs from /var/lib/docker/containers/*.
- Rename the files with the container names instead of the ids.
- Compress the log files.

```
vim /etc/logrotate.conf

/var/lib/docker/containers/*/*.log {
   copytruncate
   compress
   dateext
   daily
   dateformat -%Y%m%d%H%s
   maxsize 350M
   missingok
   olddir /var/log/docker
   sharedscripts
   lastaction
     /var/log/docker/logs_rename /var/log/docker logs
   endscript
   rotate 30
 }
```

The second step of this idea was to set up a Cron job to copy the log files to a centralized server where the logs
could be loaded and inspected.
This idea had many disadvantages and was more like reinventing the wheel. One of the most important disadvantages was
that logs would be lost if a new deployment occurred, and the containers were replaced.
That is why I started looking for an alternative solution that would help us collect the logs from the servers to a
centralized server without losing logs. Luckily, Docker supports multiple log-drivers (see the full list). One of these
log-drivers is Syslog, which is by default installed on Linux systems (no extra software needs to be installed).
For this reason, I decided to implement a centralized logging system with Syslog.

#### Implementations
For simplicity reasons only, let us assume that the infrastructure that hosts the microservices consists of the following nodes:

- Two-node Docker cluster hosting the services.
- One node for storing logs.

##### Log server configurations
In order to configure the log server and make it ready for collecting logs from the Docker hosts, I had to do the following steps:
- Make sure Syslog is installed or install it using the commands below:
```
yum update -y
yum install -y rsyslog rsyslog-doc
```
- List on the correct TCP port. The line below must exist in the Syslog config file /etc/rsyslog.conf:
```
#$ModLoad imtcp
#$InputTCPServerRun 514
```
- Update the Journal Rate limit configurations for both Syslog and journald. The configurations below should be added
to the /etc/rsyslog.conf file to avoid log loss in case of too many log messages. See Syslog’s documentation for more
information.
```
$imjournalRatelimitInterval 0
$imjournalRatelimitBurst 0
```
- In addition, Rate limit configs need to be set to 0 in thejournald configuration file /etc/systemd/journald.conf.
```
RateLimitInterval=0
```
By default, Syslog will store all logs in /var/log/messages. However, we would like to separate the logs based on the
container name to make it easier for investigating. To configure the Syslog server to separate logs from different
containers into different files, we need to perform the following steps:
- Decide and create the folder to keep all logs.
```
mkdir /var/log/docker
```
- Collect the logs for the Docker daemon and save it to the disk. The below Syslog configuration rule
(stored in /etc/rsyslog.d/docker_daemon.conf) tills Syslog to save all the logs that belong to a program that starts
docker to the /var/log/docker/daemon.log.
```
$template DockerLogs, "/var/log/docker/daemon.log"
if $programname startswith 'dockerd' then -?DockerLogs
& stop
```
- Collect Docker container logs and save them to separate log files. The Syslog configuration rule below will save the
logs into individual files for each of the running containers based on the container name. The rule should be stored in
/etc/rsyslog.d/docker_container.conf. This rule is relying on the Docker hosts to tag all the logs with container_name.
```
$template DockerContainerLogs,"/var/log/docker/%hostname%_%syslogtag:R,ERE,1,ZERO:.*container_name/([^\[]+)--end%.log"
if $syslogtag contains 'container_name'  then -?DockerContainerLogs
& stop
```
The next step is to set up a Logrotate rule to rotate the logs and avoid big log files on the server. Old logs can be
archived by system administrators. The rule below will rotate logs on the server daily or if the file size is more than
20 MB. It will also keep only the last 30 log files for each container.
```
/var/log/docker/*.log {
  copytruncate
  compress
  dateext
  size 20M
  daily
  dateformat -%Y%m%d%H%s
  missingok
  rotate 30
}
```
As a final step for server configuration, we need to restart the services to make sure our configs are loaded.
```
systemctl restart rsyslog
systemctl restart systemd-journald
```
##### Configure Docker Hosts
Now that we are done with configuring the Syslog server, we can move on and start configuring the Docker hosts to
forward the logs from the servers to the Syslog server.
The first thing that needs to be done is to reconfigure the Docker daemon to use the syslog log driver instead of
journald and to tag the logs with the container name. To achieve this goal, we need to modify the Docker daemon
configuration file that is located under /etc/docker. The daemon.json should include the following content:
```
{
  "log-driver": "syslog",
  "log-opts": {
    "syslog-address": "tcp://${SYSLOG_SERVER_IP}:514",
    "tag": "container_name/{{.Name}}",
    "labels": "${ENV_NAME}",
    "syslog-facility": "daemon"
  }
}
```
The variable SYSLOG_SERVER_IP should be replaced by the Syslog server IP. The variable ENV_NAME should be replaced by
the environment name (testing, staging, or production). With the configurations above, Docker will forward the logs
directly to the Syslog server. In case the Syslog server is down or there is no valid connection to it, we may lose
some logs with the configurations above.
To improve our logging system and avoid losing logs, we are going to do the following changes:
- Forward logs from Docker to the local server.
- Configure local Syslog to forward the logs to the centralized log server.

To forward logs from Docker to the local Syslog server, we simply need to remove the following line from
/etc/docker/daemon.json or replace SYSLOG_SERVER_IP with 127.0.0.1:
```
"syslog-address": "tcp://${SYSLOG_SERVER_IP}:514",
```
Once we are done with the Docker configuration, we need to restart the Docker daemon.
```
systemctl restart docker
```
The next step is to update the Syslog configuration on the Docker node to be able to store Docker logs locally and
forward them to the centralized server.

##### syslog and journald configurations
We need the following configuration item for journald in file /etc/systemd/journald.conf:

```RateLimitInterval=0```

We need the following configuration items for Syslog in the file /etc/rsyslog.conf:
```
$ActionQueueFileName fwdRule1
$ActionQueueSaveOnShutdown on
$ActionQueueType LinkedList
$ActionResumeRetryCount -1
$imjournalRatelimitInterval 0
$imjournalRatelimitBurst 0
```

- ActionQueueFileName: Adds unique name prefix for spool files.
- ActionQueueSaveOnShutdown: Save messages to disk on shutdown.
- ActionQueueType: Run asynchronously.
- ActionResumeRetryCount: Infinite retries if the host is down.

With the configurations above, Syslog is going to keep retrying to send logs to the centralized server until logs are
captured by the destinations server. However, until now, we have not configured the local Syslog server on the Docker
hosts to forward the logs to any other servers. Therefore, all logs collected from Docker will be saved in the default
file /var/log/messages.
Forwarding logs from Syslog to another server is very simple. You only need to add the following rule to the end of the
/etc/rsyslog.conf file and replace SYSLOG_SERVER_IP with the valid log server IP. This rule basically checks all the
logs and filters them based on the tags and the program name. If the message tags contain a tag called container_name
or programname, start with docker. Then it will forward the logs to SYSLOG_SERVER_IP.
```
if $syslogtag contains 'container_name' or $programname startswith 'docker' then @@${SYSLOG_SERVER_IP}:514
& ~
```
We can improve the rule above by keeping a copy of the logs locally on the Docker servers. With the changes below, our
Syslog rule will send the logs to the remote Syslog server, and it will keep a copy of the log files locally on the
individual servers running Docker under /var/log/docker.
```
if $syslogtag contains 'container_name' or $programname startswith 'docker' then @@${SYSLOG_SERVER_IP}:514
$template DockerContainerLogs, "/var/log/docker/%syslogtag:R,ERE,1,ZERO:.*container_name/([^\[]+)--end%.log"
if $syslogtag contains 'container_name' then -?DockerContainerLogs
& stop
```
Since we now have a copy of the log files on the Docker servers, it makes sense to add the Logrotate rule to rotate
these files too. We can use the same rule used for the server nodes.
```
/var/log/dockerlfs/*.log {
  copytruncate
  compress
  dateext
  size 20M
  daily
  dateformat -%Y%m%d%H%s
  missingok
  rotate 30
}
```
Finally, a restart is needed for both Syslog and journald to pick up the new configurations.
```
systemctl restart rsyslog
systemctl restart systemd-journald
```
Conclusion
Centralized logging systems and stacks can be implemented by many methods and tools. Syslog is one of these methods—and
it’s especially useful when there are some limitations on the running environment, and a lack of experience managing
and running more sophisticated stacks like ELK.
# docker-containers-syslog
Docker logging with rsyslog

Main goals:
Collect the logs (STDOUT/STDERR logs)  from awx containers and docker daemon beacuse I would like to ensure a smoothly operation and we need to collect and store the affected containers' log files.
We will store the log files separately and collect the logs only from that containers where - the type of logging - has been set (for the required container(s) ). That means basically, we dont collect log files from all containers. It's not necessary and we don't want to do this.
We will collect  - and store - the log files only signed containers. We are going to store log files locally and later - if needed - we can forward to remote syslog server,too.

Reason: we would like to store the log files of containers because this helps us if we have to investigate an issue


Enviroment, notes:
Everything has been validated only on Centos v7: on Centos distribution: instead of syslog you can find the relavant log records here: /var/log/messages and /var/log/secure file.
Path of collected log files - we will store the files here - : /var/log/docker/ : 
/var/log/docker/
├── containers
│   ├── common.log
│   ├── common.log-20201014081602656424.gz
│   ├── docker-awx-web.log
│   ├── docker-awx-web.log-20201017041602900121.gz
│   ├── docker-test2-nginx.log
│   ├── docker-test2-nginx.log-20201014081602656424.gz
├── daemon.log

Log records:
 from docker daemon: daemon.log
 from container: docker-awx-web.log
 from all container (all messages in one file) : common.log


Query the actual logging driver:
Globally: On docker daemon's level: docker info --format '{{.LoggingDriver}}'
-->json-file
On container's level: 
docker inspect -f '{{.HostConfig.LogConfig.Type}}' build_image_web_1
--> json-file


Configure syslog to collect log entries from docker daemon and from dedicated/signed containers:
1. Configure on system's (OS) level:
- add these to /etc/rsyslog.conf file:
$imjournalRatelimitInterval 0
$imjournalRatelimitBurst 0
- add these to /etc/systemd/journald.conf file:
RateLimitInterval=0

2. Add syslog file to system:
notes: keep in your mind if the name of next file depends on your environment, due to evaluation order!
cat 24-docker.conf |egrep -v "(^$|^#)"
$template CUSTOM_LOGS,"/var/log/docker/containers/%programname%.log"
$template DockerLogs, "/var/log/docker/daemon.log"
if $programname startswith 'dockerd' then -?DockerLogs
else
if $programname startswith 'docker-' then {
  if $syslogtag startswith 'docker-' then ?CUSTOM_LOGS
}
& ~

Validating rsyslog configuration:  rsyslogd -N1
Restart services:
 systemctl restart rsyslog
 systemctl restart systemd-journald

Set globally - we don't wannt to to this, but if you want you can - : On docker daemon's level, for example:
cat /etc/docker/daemon.json 
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "pilot1",
    "env": "multinode1"
  }
}
If you update the daemon file you need to run this command: systemctl daemon-reload&&systemctl restart docker.service  

Set on container's level:
Add syslog-support to compose file / or run the container with right parameters:
for example:
 docker run -d -p 8081:80 --name test-nginx  --log-driver=syslog --log-opt tag=test-nginx nginx
example for docker compose file:
version: '2.1'
services:
  web:
...
    logging:
      driver: syslog
      options:
        tag: docker-awx-web
    volumes:
....


Check:
check the logging driver of container:
docker inspect -f '{{.HostConfig.LogConfig.Type}}' build_image_web_1
--> syslog
check the log records:
container's log:
sudo tail -2  /var/log/docker/containers/docker-awx-web.log
Oct 15 14:57:01 awx-pilot-multinode-0001 docker-awx-web[7467]: RESULT 2
Oct 15 14:57:01 awx-pilot-multinode-0001 docker-awx-web[7467]: OKREADY
daemon's log:
sudo systemctl reload docker.service
sudo tail -2 /var/log/docker/daemon.log 
Oct 12 17:58:28 %hostname% dockerd: time="2020-10-12T17:58:28.048650186+02:00" level=info msg="ignoring event" module=libcontainerd namespace=moby topic=/tasks/delete type="*events.TaskDelete"
Oct 12 17:58:38 %hostname% dockerd: time="2020-10-12T17:58:38.298237231+02:00" level=info msg="ignoring event" module=libcontainerd namespace=moby topic=/tasks/delete type="*events.TaskDelete"


Set the logrotation on new,generated log files: if needed edit this based on your espectations, like "size" parameter
cat /etc/logrotate.d/docker_syslog 
/var/log/docker/*.log
/var/log/docker/containers/*.log
{
  copytruncate
  compress
  dateext
  size 1M
  daily
  dateformat -%Y%m%d%H%s
  missingok
  rotate 30
}

Check: logrotate --force docker_syslog

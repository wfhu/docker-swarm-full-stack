# docker-swarm-full-stack
Buildup a docker swarm cluster with opensource tools for production environment  
使用开源工具，从零开始搭建完整生态的swarm mode生产环境集群


## Prerequisite
* you should be able to setup a working ElasticSearch serivce, and a Kibana page
* you'd better have created your own docker image registry
* you need root access to the node servers, CentOS 7.3 is used within this demo


## Using the following tools:  
## 主要使用以下工具：

cluster management and Orchestration : Docker Swarm Mode  
container monitor : cAdvisor + prometheus  
node monitor : node_exporter + prometheus  
monitor display : grafana  
cluster management UI : portainer    
log collection & display & search  : ELK+logspout  

集群管理和编排：Docker Swarm Mode   
容器监控：cAdvisor + prometheus    
节点监控：node_exporter + prometheus  
监控展示：grafana    
前端UI界面：portainer  
日志搜集展示和搜索：ELK+logspout    



## Part One, let us setup the Swarm cluster   
## 首先，我们先搭建集群  

### 1. install the docker-ce tools

```
yum install -y yum-utils device-mapper-persistent-data lvm2

yum-config-manager --enable extras

yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum makecache fast

yum install -y docker-ce

systemctl start docker

systemctl enable docker

docker run hello-world

```



### 2. setup the swarm manager leader node
```
docker  swarm init --advertise-addr 192.168.33.5
```



### 3. add two manager node, join this swarm as manager node(also as worker node)
```
docker swarm join --token SWMTKN-1-YOUR-MANAGER-TOKEN 192.168.33.5:2377
```



### 4. add two worker node
```
docker swarm join --token SWMTKN-1-YOUR-WORKER-TOKEN 192.168.33.5:2377

```






## Part Two, let start the monitor agents:  
## 然后，我们需要把监控的agent进程起来 

### 1. use cAdvisor to monitor container's CPU/Memory/Network    
```
docker service create --name cadvisor \
    --mount type=bind,source=/var/lib/docker/,destination=/var/lib/docker,readonly \
    --mount type=bind,source=/var/run,destination=/var/run \
    --mount type=bind,source=/sys,destination=/sys,readonly \
    --mount type=bind,source=/,destination=/rootfs,readonly \
    --mode global \
    --detach=true \
    --publish mode=host,published=18080,target=8080 \
    google/cadvisor:latest
```


### 2. use prometheus's node_exporter to monitor Swarm node's basic infomation     
```
docker service create --name node_exporter \
    --mount type=bind,source=/proc,destination=/host/proc,readonly \
    --mount type=bind,source=/sys,destination=/host/sys,readonly \
    --mount type=bind,source=/,destination=/rootfs,readonly \
    --mode global \
    --detach=true \
    --publish mode=host,published=9100,target=9100 \
    quay.io/prometheus/node-exporter  \
    -collector.procfs /host/proc \
    -collector.sysfs /host/sys \
    -collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"
```

### 3. configure the prometheus server, add the above newly added targets    
```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.

  - job_name: 'MySwarmCluster'
    static_configs:
      - targets: ['192.168.24.160:18080','192.168.24.160:9100'] 
```


After this, using my grafana template, you could see the Swarm cluster and Services running in your Swarm cluster  

![grafana Docker Swarm Dashboard](/images/grafana_template_screenshot.jpg)



## Part Three, we need try to setup the portainer management UI

### first we need to reconfigure docker-engine to liston on TCP address, other than the UNIX socket
```
sed -i "/^ExecStart/c ExecStart=/usr/bin/dockerd -H unix:///var/run/docker.sock -H tcp://$(ip a |grep global |grep eth0 |awk '{print $2}' |cut -d'/' -f1):2375" /usr/lib/systemd/system/docker.service
grep '^ExecStart' /usr/lib/systemd/system/docker.service
systemctl daemon-reload
systemctl restart docker
```

### Now, make a directory and start the portainer
```
mkdir -p /data/portainer_prod

docker service create \
    --name portainer \
    --publish 9000:9000 \
    --constraint 'node.role == manager' \
    --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
    --mount type=bind,src=/data/portainer_prod,dst=/data \
    portainer/portainer \
    -H unix:///var/run/docker.sock
````

Then, you can visit http://your-ip-address:9000 to visit the portainer UI



## Part Four, let start to collect stdout log and transfer the log to ElasticSearch

log stream like this : *container stdout* -> *logspout in each node* -> *logstash inside the Cluster* -> *outside ElasticSearch*

### 1. create a new overlay network for the log transfer, as logspout need to communicate with logstash
```
docker network create --driver overlay lognet
```

### 2. create the logstash service, listening on TCP/19300 port
```
docker service create \
    --name mylogstash \
    --network lognet \
    --publish 19300:19300 \
    logstash -e 'input { tcp { port => 19300 mode => "server" ssl_enable => false } } output { elasticsearch { hosts => ["YOUR-ElasticSearch-ADDRESS:PORT"] index => "my-docker-cluster"} }'
```

### 3. create the logspout service

#### build your own image from *logspout*, add support to logstash
```
cd mylogspout && docker build -t mylogspout:v1 .
```
if there's error, please make sure mylogspout/build.sh file has exec bit set, check [here](https://github.com/gliderlabs/logspout/issues/238) for more information

#### then tag/push your locally build image to YOUR-REGISTRY
```
# docker login -u YOUR-USER-NAME -p YOUR-PASSWORD  YOUR-REGISTRY-ADDRESS
# docker tag mylogspout:v1 YOUR-REGISTRY-ADDRESS/mylogspout:v1
# docker push YOUR-REGISTRY-ADDRESS/mylogspout:v1
```

#### now you can create your *logsplout* service based on the new image
```
docker service create \
    --name mylogspout \
    --network lognet \
    --with-registry-auth \
    --mode global \
    --detach=true \
    --mount type=bind,source=/var/run/docker.sock,destination=/var/run/docker.sock,readonly \
    -e ROUTE_URIS=logstash+tcp://mylogstash:19300 \
    YOUR-REGISTRY-ADDRESS/mylogspout:v1 
```

*If your containers just print all logs to stdout/stderr, you'd better change the *[max-size](https://docs.docker.com/engine/admin/logging/json-file/)* of the JSON File log driver*







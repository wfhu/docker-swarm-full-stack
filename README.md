# docker-swarm-full-stack
Buildup a docker swarm cluster with opensource tools for production environment  
使用开源工具，从零开始搭建完整生态的swarm生成环境集群

Using the following tools:  
主要使用以下工具：

container management and Orchestration : Docker Swarm Mode  
container monitor & display : cAdvisor(https://github.com/google/cadvisor) + prometheus(https://prometheus.io/) + grafana(https://grafana.com/)  
node monitor & display : node_exporter(https://github.com/prometheus/node_exporter) + prometheus + grafana  
UI : portainer   
log collection & display & search  : ELK +   

集群管理和编排：Docker Swarm Mode   
容器监控和展示：cAdvisor(https://github.com/google/cadvisor) + prometheus(https://prometheus.io/) + grafana(https://grafana.com/)   
节点监控和展示：node_exporter(https://github.com/prometheus/node_exporter) + prometheus + grafana   
前端UI界面：portainer    
日志搜集展示和搜索：ELK   



First, let us setup the Swarm cluster   
首先，我们先搭建集群  

1. install the docker-ce tools

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



2. setup the swarm manager leader node
```
docker  swarm init --advertise-addr 192.168.33.5
```



3. add two manager node, join this swarm as manager node(also as worker node)
```
docker swarm join --token SWMTKN-1-YOUR-MANAGER-TOKEN 192.168.33.5:2377
```



4. add two worker node
```
docker swarm join --token SWMTKN-1-YOUR-WORKER-TOKEN 192.168.33.5:2377

```






Now, let start the monitor agents:  
然后，我们需要把监控的agent进程起来 

1. use cAdvisor to monitor container's CPU/Memory/Network  
使用cAdvisor监控容器内的信息，主要包括CPU、内存、网络   
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


2. use prometheus's node_exporter to monitor Swarm node's basic infomation   
使用prometheus node_exporter监控Swarm集群节点的基本信息   
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

3. configure the prometheus server, add the above newly added targets  
配置prometheus服务，添加以上监控目标   
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

![grafana Docker Swarm Dashboard](/images/grafana.jpg)



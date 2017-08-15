# docker-swarm-full-stack
Buildup a docker swarm cluster with opensource tools for production environment



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

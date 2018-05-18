
# docker swarm and registry
---
## 创建
manager 节点
```sh
$ docker swarm init --advertise-addr 192.168.99.100
Swarm initialized: current node (dxn1zf6l61qsb1josjja83ngz) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
work节点
```sh
$ docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377
```
## [常用命令](https://docs.docker.com/engine/swarm/#feature-highlights)
```sh
docker swarm init
docker swarm join
docker node ls
docker service create
docker service inspect
docker service ls
docker service rm
docker service scale
docker service ps
docker service update
```
## 使用tls搭建私人registry
```sh
生成密码
 $ docker run   --entrypoint htpasswd   registry:2 -Bbn testuser testpassword > auth/htpasswd

生成key
 $ openssl req -newkey rsa:2048 -nodes -sha256 -keyout certs/domain.key -x509 -days 365 -out certs/domain.crt
 
启动registry 
 $ docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v `pwd`/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v `pwd`/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2

配置hosts
192.168.1.223 mydockerhub.com
使用证书
 $ cp ./certs/domain.crt /etc/docker/certs.d/mydockerhub.com\:5000/ca.crt 
```

## 在swarm下使用私人registry
swarm所有节点需要配置hosts
```sh
$ docker login registry.example.com

$ docker service  create \
  --with-registry-auth \
  --name my_service \
  registry.example.com/acme/my_image:latest
```
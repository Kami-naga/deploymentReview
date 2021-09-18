# Deployment Review
try to deploy a project

## Create a virtual machine 
Here I used vmware workstation 16 + ubuntu18.04 to create my ubuntu VM and used Xshell to get access to it(XFTP to transfer files).

See `https://www.codetd.com/ja/article/12040939`

- here I found my VM can not get access to Internet.
    - see the link in issue 1
- in order to let it accessible by ssh, we need to install openssh-server for it
    - `sudo apt-get install openssh-server`
    - `ps -e |grep ssh`
    - if you find sshd, it's OK
- `ip address` to get the address of the vm
- make a new session in XShell and connect to the ip address above(port is 22), then you can now use xshell to control the VM elsewhere(the configuration of xftp is the same as xshell)
- firewall settings (expose the port you need)
    - See `https://linuxconfig.org/how-to-install-and-use-ufw-firewall-on-linux#h3-3-centos`
    - `https://blog.csdn.net/qq_15256443/article/details/101289412`
## Get a project to deploy
A project called renren-fast

(backend：spring+spring mvc+mybatis， frontend:VUE + Element UI)

See `https://www.renren.io/guide/`

Install the dependency & run
(node version needs to be 8.x)

## Docker Commands
- `docker search xxx`
  - then `docker pull yyy`
- see images: `docker images`
- see containers: `docker ps -a` 
- To a File: `docker save yyy > PATH(e.g. /home/zzz.tar.gz)`
  - `docker load < PATH(/home/zzz.tar.gz)`
- change the name of the image `docker tag yyy ccc`
  - then you can find 2 images, 1 with the name yyy and 1 with the name ccc
  - just delete the yyy image 
- delete the image: `docker rmi yyy`
- `docker run yyy bash`
  - `-it` interation mode
  - `-d` run in background
  - `--name aaa` give a name: aaa
  - `-p` port mapping host:container(if several ports need to be exposed, use several -p)
  - `-v` dir mapping host:container
  - `--privileged` give permissions to rwx the file
  - use `exit` to exit interation mode(also stop the container)
- `docker start -i aaa` restart the container with interaction
- `docker pause aaa`
- `docker unpause aaa`
- `docker stop aaa`
- the container can be deleted by `docker rm aaa` after you stopped it.
- if have any problem when running, use `docker logs aaa` to see the logs
## Create the MySQL cluster
2 kinds of cluster: replication & pxc

Since the data is somewhat important, so use pxc to ensure strong consistency.

*replication way can see `https://blog.csdn.net/gu_wen_jie/article/details/102721524`

Get the cluster image with name "pxc" by following cmds:
  - `docker pull percona/percona-xtradb-cluster:5.7.21`
  - `docker tag percona/percona-xtradb-cluster:5.7.21 pxc`
  - `docker rmi percona/percona-xtradb-cluster:5.7.21`

For safety reasons, we need to create a Docker internal network for pxc cluster instances, like
- `docker network create --subnet=172.18.0.0/24 net1`
- `docker network inspect net1`
- `docker network rm net1`
- `docker network ls`

For data saving, PXC can not use bind mount, so it needs a docker volume
  - bind mount is not portable across different host systems(This is why bind mount cannot appear in a Dockerfile)
  - first create a volume: `docker volume create --name v1`
  - `docker vulume ls`
  - `docker inspect v1`
  - `docker volume rm v1`
  - then map the volume to a container

Use the following cmd to create pxc container

first one:

`docker run -d -p 3306:3306 -v v1(docker volume name):/var/lib/mysql --privileged --name=node1 --net=net1 --ip 172.18.0.2 -e MYSQL_ROOT_PASSWORD=xxxx -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=xxxx pxc`

others: 
port,volume,name,ip should be changed & add `-e CLUSTER_JOIN=node1`

e.g.  `docker run -d -p 3307:3306 -v v2:/var/lib/mysql --privileged --name=node2 --net=net1 --ip 172.18.0.3 -e MYSQL_ROOT_PASSWORD=xxxx -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=xxxx -e CLUSTER_JOIN=node1 pxc`

*Since the support for TLSv1 & TLSv1.1 was dropped, when connecting to those dbs, we need to add `?enabledTLSProtocols=TLSv1.2` at the end of the URL.

*if you use DataGrip to monitor the database status, you should go to properties->schema and tick all schemas , then you can see all schemas' changes.(you can only see the the schema you ticked in the database)

## Database load balancing
Use Haproxy
- `docker pull haproxy`

See `https://www.percona.com/doc/percona-xtradb-cluster/LATEST/howtos/haproxy.html` to learn how to configure the haproxy
- admin -> 8888 (user:admin, pw:123)
- create an account without permissions(name can be haproxy, no pw) in each mySQL to get the heartbeat 
  - `CREATE USER 'haproxy'@'%' IDENTIFIED BY '';`

create container
- `docker run -itd -p 4001:8888 -p 4002:3306 -v /home/soft/haproxy:/usr/local/etc/haproxy --name h1 --privileged --net=net1 haproxy bash`

then get into the container and apply the cfg file
- `docker exec -it -u root h1 bash`
  - cmd below needs permission so get into the container as the root user
- `haproxy -f /usr/local/etc/haproxy/haproxy.cfg`
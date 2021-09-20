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
## Create MySQL cluster
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
  - but bind mount can do file mapping which volume cannot do(volume can do dir mapping only)
  - first create a volume: `docker volume create --name v1`
  - `docker vulume ls`
  - we can use cmd `docker inspect v1` to know the path(mountpoint) of the volume
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

*suspend(not shut down) the virtual machine to ensure that the container runs properly when it is started again. However, after the suspension, the docker container will not be able to gain access to the network when the virtual machine resumes , so you need to change the configuration
- `vim /etc/sysctl.conf`
- add `net.ipv4.ip_forward=1`
- `sudo service network-manager restart` (CentOS7.x- :`systemctl restart network`)
- `sudo service keepalived restart`
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

High Availability

Use keepalived to implement dual-system.

See `https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/7/html/load_balancer_administration/ch-initial-setup-vsa`
- instance
  - install keepalived in haproxy container
    - `docker exec -it -u root h1 bash`
    - `apt-get update`
    - `apt-get install keepalived`
  - configure keepalived
    -  configure keepalived.conf 
    - add keppalived config file to bind mount
      - so the docker run cmd changed:
        - `docker run -itd -p 4001:8888 -p 4002:3306 -v /home/soft/h1/haproxy:/usr/local/etc/haproxy -v /home/soft/h1/keepalived:/etc/keepalived --name h1 --privileged --net=net1 haproxy bash`
  - `service keepalived start`
    - check by cmd: 
      - `ping 172.18.0.201(the virtual IP you wrote in the conf file)`

  - get another haproxy+keepalived container
    - `docker run -itd -p 4003:8888 -p 4004:3306 -v /home/soft/h2/haproxy:/usr/local/etc/haproxy -v /home/soft/h2/keepalived:/etc/keepalived --name h2 --privileged --net=net1 haproxy bash`
    - ...
- server
  - `apt-get update`
  - `apt-get install keepalived`
  - configure keepalived.conf(put it in path: /etc/keepalived)
  - `service keepalived start`

## Implement hot backup
- cold backup 
  - mysqldump or just copy the data file of mysql
- hot backup
  - LVM(linux backup approach, all db ok)
    - read only & can not write
  - XtraBackup(mysql)
    - no lock!
    - do not interrupt the transaction
    - compressed

- full backup
- incremental backup
- differential backup

XtraBackup uses full + incremental

- first create backup volume
  - `docker volume create backup`
- then stop & delete one of the db nodes(here delete node1)
- start it with backup
  - `docker run -d -p 3306:3306 -v v1:/var/lib/mysql -v backup:/data --privileged --name=node1 --net=net1 --ip 172.18.0.2 -e MYSQL_ROOT_PASSWORD=xxxx -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=xxxx -e CLUSTER_JOIN=node2 pxc`
- `docker exec -it -u root node1 bash`
- `apt-get update`
- `apt-get install percona-xtrabackup-24`
- do full backup: `innobackupex --user=root --password=xxxx /data/backup/full`
- we can see the path of the backup file is in`/data/backup/full/2021-09-19_17-13-38/`
- how to restore?
  - we can only cold restore, no hot restore(we can not write new data while restoring data)
  - for pxc cluster, it's a bit more complicated
    - other nodes do not know how to sync data with the restored node
    - we have to dissolve the cluster and then restore the node data(rollback the uncommitted transaction before restoration), then let other nodes join the new cluster to sync the data
      - `docker stop node2,3...` & `docker rm node2,3...`
      - delete all data: `rm -rf /var/lib/mysql/*`
      - rollback transaction: `innobackupex --user=root --password=xxxx --apply-back /data/backup/full/2021-09-19_17-13-38/`
      - restore: `innobackupex --user=root --password=xxxx --copy-back  /data/backup/full/2021-09-19_17-13-38/`

## Create Redis cluster
- See `https://redis.io/topics/cluster-tutorial`
- `docker network create --subnet=172.19.0.0/16 net2`
- `docker pull redis`(`docker pull yyyyttttwwww/redis` which has its config file modified)
- `docker run -itd --name r1 -p 5001:6379 --net=net2 --ip 172.19.0.2 redis bash`
- `docker run -it -d --name r2 -p 5002:6379 --net=net2 --ip 172.19.0.3 redis bash`
- ...(6 redis = 3x(master+slave))
- `docker exec -it -u root r1 bash`
- modify the configuration to fit the cluster(`/usr/redis/redis.conf`)
  - `daemonize yes` -> to background
  - `cluster-enabled yes` -> cluster mode on
  - `cluster-config-file nodes.conf` -> cluster configuration file
  - `cluster-node-timeout 15000`
  - `appendonly yes` -> change the persistence mode to AOF mode
   - 2 modes (AOF & RDB(default))
    - RDB: high efficiency but low reliability 
    - AOF: high reliability but low efficiency
- `cd /usr/redis/src`
- `./redis-server ../redis.conf`
- do the above things in r2~r6(if you use yyyyttttwwww/redis, only do above 2 line in r2-r6)
- since the redis version is a bit low, we need a script redis-trib.rb which is in src dir to create cluster(high version can use redis-cli)
  - `apt-get install ruby`
  - `apt-get install rubygems`
  - `gem install redis`
  - `cd /usr/redis/src`
  - `mkdir ../cluster`
  - `cp /usr/redis/src/redis-trib.rb /usr/redis/cluster/`
  - `cd ../cluster`
  - `./redis-trib.rb create --replicas 1 172.19.0.2:6379 172.19.0.3:6379 172.19.0.4:6379 172.19.0.5:6379 172.19.0.6:6379 172.19.0.7:6379`
  - `/usr/redis/src/redis-cli -c` to start a redis client
    - `set a 10` & `get a 10`
    - pause one of the redis & see `cluster nodes` in one redis client

## Backend Deployment
- create a schema called `renren_fast`
- execute the script(/db/mysql.sql) to get the table(through datagrip) 
- change the ip address of mysql in application-dev.yml & application-prod.yml
- change the config in application.yml
  - configure redis cluster
    - comment out the host, port & password
    - add  `cluster:nodes:- 172.19.0.2:6379- 172.19.0.3:6379- 172.19.0.4:6379 - 172.19.0.5:6379 - 172.19.0.6:6379 - 172.19.0.7:6379`
  - change the port of tomcat: 8080 -> 6001
    - nodes in different docker net can not access each other, how can the backend node get access to mysql(net1) & redis(net2)?
      - let backend node use the host's net & change make each backend have different port number(6001, 6002...)
- get jar file of the backend(2 times)
  - change server port & `mvn clean install -Dmaven.test.skip=true`
- `docker volume create j1` & `docker volume create j2` &upload jar pack by XFTP
- `docker run -itd --name j1 -v j1:/home/soft --net=host java:8` & `docker run -itd --name j2 -v j2:/home/soft --net=host java:8`
- `docker exec -it j1 bash`
- `nohup java -jar /home/soft/renren-fast.jar`

Use Nginx to do load balancing
- `docker pull nginx`
- core part of the configuration can see `https://www.nginx.com/resources/wiki/start/topics/examples/loadbalanceexample/`
- configure keepalived
- `docker run -itd --name n1 -v /home/soft/blb-n1/nginx/nginx.conf:/etc/nginx/nginx.conf -v /home/soft/blb-n1/keepalived:/etc/keepalived --net=host --privileged nginx bash`
- `docker exec -it -u root n1 bash`
- `apt-get update`
- `apt-get install keepalived`
- `service nginx start`
- `service keepalived start`
  - check it by ping at the host machine
- get the second nginx for backend load balancing
- now 2 backends & 2 nginx all get better availability

## Frontend Deployment

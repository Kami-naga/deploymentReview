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


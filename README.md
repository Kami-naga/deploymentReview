# Deployment Review
try to deploy a project

## Create a virtual machine 
Here I used vmware workstation 16 + ubuntu18.04 to create my ubuntu VM and used Xshell to get access to it(Also XFTP to transfer files).

See `https://www.codetd.com/ja/article/12040939`

- here I found my VM can not get access to Internet.
    - see the link in issue 1
- in order to let it accessible by ssh, we need to install openssh-server for it
    - `sudo apt-get install openssh-server`
    - `ps -e |grep ssh`
    - if you find sshd, it's OK
- `ip address` to get the address of the vm
- make a new session in XShell and connect to the ip address above(port is 22), then you can now use xshell to control the VM elsewhere(the operation of xftp is the same as xshell)
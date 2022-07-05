# k8s-installer

## state 1 :

1 master node 

3 worker nodes

### Requirment :

#### Software requirment :
4 vms with ubuntu 18.04 or ubuntu 20.04

#### Hardware requirment :
at least 2GB Memory and 2Core CPU with 20GB HDD

in this tutorial we have 3 nodes that they have internet access and they have private ip address:

>> master 192.168.1.3
> 
> worker1 192.168.1.4
> 
> worker2 192.168.1.5

all of nodes should have root access with password.
we will install ansible on master node so it should has ssh access to all of the worker nodes.
before start you shoud edit hosts files and add these lines end of the
#### edit hosts file with nano text editor
>nano /etc/hosts
#### add these lines to end of file
>192.168.1.3 master
>
>192.168.1.4 worker1
>
>192.168.1.5 worker2

#### create ssh access from master node to workers
>ssh-keygen
>
>Enter
>
>Enter


>ssh-copy-id root@192.168.1.3
>
>yes
>
>type root password


>ssh-copy-id root@192.168.1.4
>yes
>
>type root password
>

>ssh-copy-id root@192.168.1.5
>
>yes
>
>type root password

#### in this tutorial we will use from ansible and playbook so we need to install ansible on master node
>sudo apt update
>
>sudo apt-get install ansible

#### create directory in master node
>mkdir ~/kube-cluster
>
>cd ~/kube-cluster

#### create and open hosts file in this directory 
>nano ~/kube-cluster/hosts

#### add these lines to hosts file:
>[masters]
>
>master ansible_host=192.168.1.3 ansible_user=root
>
>[workers]
>
>worker1 ansible_host=192.168.1.4 ansible_user=root
>
>worker2 ansible_host=192.168.1.5 ansible_user=root
>
>[all:vars]
>
>ansible_python_interpreter=/usr/bin/python3

#### now we should test to make sure that everything work perfect
>ansible all -m ping -u root
#### now you should see this output in terminal:
>master | SUCCESS => 
>{
>    "changed": false, 
>    
>    "ping": "pong"
>
>}
>worker1 | SUCCESS => 
>{
>    "changed": false,
>     
>    "ping": "pong"
>
>}
>worker2 | SUCCESS => 
>{
>    "changed": false, 
>    
>    "ping": "pong"
>
>}

#### we need to create a sudo user (passwordless) in all of node so create a new file **initial.yml** with this content:

>- hosts: all
>
>  become: yes
>  
>  tasks:
>  
>    - name: create the 'ubuntu' user
>    
>      user: name=ubuntu append=yes state=present createhome=yes shell=/bin/bash
>      
>
>    - name: allow 'ubuntu' to have passwordless sudo
>    
>      lineinfile:
>      
>        dest: /etc/sudoers
>        
>        line: 'ubuntu ALL=(ALL) NOPASSWD: ALL'
>        
>        validate: 'visudo -cf %s'
>        
>
>    - name: set up authorized keys for the ubuntu user
>    
>      authorized_key: user=ubuntu key="{{item}}"
>      
>      with_file:
>      
>        - ~/.ssh/id_rsa.pub
>        

# k8s-installer

## State 1 (Single Control Plane) :

1 master node as a control plane

2 worker nodes

### Requirment :

#### Software requirment :
3 VMs with Ubuntu 18.04 or ubuntu 20.04

#### Hardware requirment :
at least 2GB Memory and 2Cores CPU with 20GB HDD

in this tutorial we have 3 nodes that they have internet access and they have private ip address:

><pre>master 192.168.1.3
> 
> worker1 192.168.1.4
> 
> worker2 192.168.1.5</pre>

all of nodes should have root access with password.
we will install ansible on master node so it should has ssh access to all of the worker nodes.
before start you shoud edit hosts files and add these lines end of them
#### edit hosts file with nano text editor
```ssh
nano /etc/hosts
```
#### add these lines to end of file
><pre>192.168.1.3 master
>
>192.168.1.4 worker1
>
>192.168.1.5 worker2</pre>

#### create ssh access from master node to workers
```ssh
ssh-keygen
```
><pre>
>Enter
>
>Enter</pre>


```ssh 
ssh-copy-id root@192.168.1.3
```
><pre>yes
>
>type root password</pre>


```ssh
ssh-copy-id root@192.168.1.4
```
><pre>yes
>
>type root password</pre>

```ssh
ssh-copy-id root@192.168.1.5
```
><pre>
>yes
>
>type root password</pre>

#### in this tutorial we will use from ansible and playbook so we need to install ansible on master node
```ssh
sudo apt update

sudo apt-get install ansible
```
#### create directory in master node
```ssh
mkdir ~/kube-cluster

cd ~/kube-cluster</pre>
```
#### create and open hosts file in this directory as an ansible inventory host
```ssh
nano ~/kube-cluster/hosts
```

#### add these lines to hosts file:
><pre>[masters]
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
>ansible_python_interpreter=/usr/bin/python3</pre>

#### now you can test it to make sure that everything work perfect!
```ssh
ansible -i hosts all -m ping -u root
```
#### now you should see this output in terminal:
><pre>master | SUCCESS => 
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
>}<\pre>

#### we need to create a sudo user (passwordless) in all of nodes so create a new file initial.yml with this content:

><pre>- hosts: all
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
></pre>

#### run the playbook with this command:
```ssh
ansible-playbook -i hosts ~/kube-cluster/initial.yml
```

#### now we nead to install some dependencies in all of nodes so create a new file kube-dependencies.yml with this content:
 
><pre> - hosts: all
>  become: yes
>  tasks:
>   - name: install Docker
>     apt:
>       name: docker.io
>       state: present
>       update_cache: true
>
>   - name: install APT Transport HTTPS
>     apt:
>       name: apt-transport-https
>       state: present
>
>   - name: add Kubernetes apt-key
>     apt_key:
>       url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
>       state: present
>
>   - name: add Kubernetes' APT repository
>     apt_repository:
>      repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
>      state: present
>      filename: 'kubernetes'
>
>   - name: install kubelet
>     apt:
>       name: kubelet=1.24.2-00
>       state: present
>       update_cache: true
>
>   - name: install kubeadm
>     apt:
>       name: kubeadm=1.24.2-00
>       state: present
>
>- hosts: master
>  become: yes
>  tasks:
>   - name: install kubectl
>     apt:
>       name: kubectl=1.24.2-00
>       state: present
>       force: yes</pre>

#### run the playbook with this command:
```ssh
ansible-playbook -i hosts ~/kube-cluster/kube-dependencies.yml
```

#### we need to use from kubeadm to create pod network and initialize the cluster on master node so crate a new file master.yml with this content:

><pre>- hosts: master
>  become: yes
>  tasks:
>    - name: initialize the cluster
>      shell: kubeadm init --pod-network-cidr=10.244.0.0/16 >> cluster_initialized.txt
>      args:
>        chdir: $HOME
>        creates: cluster_initialized.txt
>
>    - name: create .kube directory
>      become: yes
>      become_user: ubuntu
>      file:
>        path: $HOME/.kube
>        state: directory
>        mode: 0755
>
>    - name: copy admin.conf to user's kube config
>      copy:
>        src: /etc/kubernetes/admin.conf
>        dest: /home/ubuntu/.kube/config
>        remote_src: yes
>        owner: ubuntu
>
>    - name: install Pod network
>      become: yes
>      become_user: ubuntu
>      shell: kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml >> pod_network_setup.txt
>      args:
>        chdir: $HOME
>        creates: pod_network_setup.txt</pre>

#### we use from flannel for networking in k8s . you can use from other solution such as Calico or other third party network's drivers .
#### run playbook with this command:
```ssh
ansible-playbook -i hosts ~/kube-cluster/master.yml
```

after that we need to prepare workers so create a new file workers.yml with this content:

><pre>- hosts: master
>  become: yes
>  gather_facts: false
>  tasks:
>    - name: get join command
>      shell: kubeadm token create --print-join-command
>      register: join_command_raw
>
>    - name: set join command
>      set_fact:
>        join_command: "{{ join_command_raw.stdout_lines[0] }}"
>
>
>- hosts: workers
>  become: yes
>  tasks:
>    - name: join cluster
>      shell: "{{ hostvars['master'].join_command }} >> node_joined.txt"
>      args:
>        chdir: $HOME
>        creates: node_joined.txt</pre>

#### run playbook with this command:
```ssh
ansible-playbook -i hosts ~/kube-cluster/worker.yml
```

# Test
#### now switch user to ubuntu in master node or login with ubuntu user:

```ssh
ssh ubuntu@192.168.1.3
```

#### then run kubectl to see status:

```sh
kubectl get nodes
```

#### result should be as same as below:

><pre>NAME      STATUS   ROLES           AGE     VERSION
>master    Ready    control-plane   2m36s   v1.24.2
>worker1   Ready    <none>          47s     v1.24.2
>worker2   Ready    <none>          47s     v1.24.2
></pre>

#### now we can deploy a project with k8s cluster.for example for nginx you can run these commands on master node:

```sh
kubectl create deployment nginx --image=nginx
kubectl expose deploy nginx --port 80 --target-port 80 --type NodePort
```
#### you can confirm that nginx is runnig on workers with this command:
```ssh
kubectl get services
```

## State 2 (Multiple Control Plane with HAproxy) :

in this state we should have at least 7 VMs .

1 VM for installing and configuring ansibe , haproxy , kubectl , NAT and DHCP server. we use from this vm as a "gateway" for all other nodes.  

3 VMs for masters (the number of masters should be ODD for example 3,5,7 ...)

3 VMs for workers

masters and workers have only 1 network interface and connected together in local network. 

"gateway" node has 2 network interfaces . first NIC (for example eth0) is in local network and another NIC (for example eth1) have Public IP Address.

All of VMs should have Clean OS with unique Mac Address .We suggest you to install OS from the lastest realease . use from ISO to install OS and do not use from any templates .

In this case we use ubuntu 20.04.

Main VM (Haproxy , dhcp ,...) - OS = Ubuntu 20.04 - 2Core CPU - 2GB Memory - 20GB HDD - (eth0 : 192.168.1.100 - eth1 : WAN IP ) - Hostname ha)

master1 VM - OS = Ubuntu 20.04 - 2Core CPU - 2GB Memory - 20GB HDD - (eth0 : 192.168.1.101) - Hostname master1)

master2 VM - OS = Ubuntu 20.04 - 2Core CPU - 2GB Memory - 20GB HDD - (eth0 : 192.168.1.102) - Hostname master2)

master3 VM - OS = Ubuntu 20.04 - 2Core CPU - 2GB Memory - 20GB HDD - (eth0 : 192.168.1.103) - Hostname master3)

worker1 VM - OS = Ubuntu 20.04 - 2Core CPU - 2GB Memory - 20GB HDD - (eth0 : 192.168.1.104) - Hostname worker1)

worker2 VM - OS = Ubuntu 20.04 - 2Core CPU - 2GB Memory - 20GB HDD - (eth0 : 192.168.1.105) - Hostname worker2)

worker3 VM - OS = Ubuntu 20.04 - 2Core CPU - 2GB Memory - 20GB HDD - (eth0 : 192.168.1.106) - Hostname worker3)

you can have any number of workers or masters at the begin of this state. ip addresses and dhcp config may be changed base on your situation.

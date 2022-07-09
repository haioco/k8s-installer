# k8s-installer

## State 1 (Single Control Plane) :

### Requirment :

#### Software requirment :
3 VMs with Ubuntu 18.04 or ubuntu 20.04

#### Hardware requirment :
at least 2GB Memory and 2Cores CPU with 20GB HDD

#### to initialize VMs run the the initial playbook with this command:

```ssh
ansible-playbook -i hosts ~/kube-cluster/initial.yml
```
#### to install dependencies on all of nodes run the kube-dependencies playbook with this command:
```ssh
ansible-playbook -i hosts ~/kube-cluster/kube-dependencies.yml
```
#### we use from flannel for networking in k8s . you can use from other solution such as Calico or other third party network's drivers .
#### to prepare master node run master playbook with this command:
```ssh
ansible-playbook -i hosts ~/kube-cluster/master.yml
```

#### to join worker nodes to cluster run playbook with this command:
```ssh
ansible-playbook -i hosts ~/kube-cluster/workers.yml
```
# Test
#### now switch user to ubuntu user in master node or login with ubuntu user:
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
<hr>
## State 2 (Multiple Control Plane with HAproxy) :

###in this state we should have at least 7 VMs .

#### 1 VM for installing and configuring ansibe , haproxy , kubectl , NAT and DHCP server. we use from this vm as a "gateway" for all other nodes.

#### 3 VMs for masters (the number of masters should be ODD for example 3,5,7 ...)

#### 3 VMs for workers

#### to initialize VMs run the the initial playbook with this command:

```ssh
ansible-playbook -i hosts ~/kube-cluster/initial.yml
```
#### to install dependencies on all of nodes run the kube-dependencies playbook with this command:

```ssh
ansible-playbook -i hosts  --limit '!ha' ~/kube-cluster/kube-dependencies.yml
```
#### we use from flannel for networking in k8s . you can use from other solution such as Calico or other third party network's drivers .
#### to prepare master1 node run master playbook with this command:

```ssh
ansible-playbook -i hosts ~/kube-cluster/master.yml
```

#### to join other masters nodes to cluster run the playbook:
```ssh
ansible-playbook -i hosts ~/kube-cluster/cmaster.yml
```  
#### to join worker nodes to cluster run playbook with this command:
       
```ssh
ansible-playbook -i hosts ~/kube-cluster/workers.yml
```
#### in this scenario we use from ha node to manage cluster.
#### to copy cluster config file to ha node run cp-config playbook with this command:
       
```ssh
ansible-playbook -i hosts ~/kube-cluster/cp-config.yml
```
#### install lastest version of kubectl on ha node:
     
```ssh
wget https://storage.googleapis.com/kubernetes-release/release/v1.24.2/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin
```
#### run kubectl to see status:
        
```sh
kubectl get nodes
```
#### result should be as same as below:
        
><pre>
>NAME      STATUS   ROLES           AGE     VERSION
>master1   Ready    control-plane   1h   v1.24.2
>master2   Ready    control-plane   1h   v1.24.2
>master3   Ready    control-plane   1h   v1.24.2
>worker1   Ready    <none>          1h   v1.24.2
>worker2   Ready    <none>          1h   v1.24.2
>worker3   Ready    <none>          1h   v1.24.2
></pre>

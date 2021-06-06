# ansible-k8s-setup
This will setup a kubernetes cluster on Centos7 machines using ansible.
You need these machines:
1. Ansible controller - controller.example.com - 10.0.0.99 - 1 vcpu / 2 gib ram
2. Kubernetes Master - master.example.com - 10.0.0.100 - 2 vcpu / 4 gib ram
3. Kubernetes Node1 - nodeone.example.com - 10.0.0.1 - 1 vcpu / 4 gib ram
4. Kubernetes Node2 - nodetwo.example.com - 10.0.0.2 - 1 vcpu / 4 gib ram

If you can allocate more compute resources, its better
If you change your machine IP's then you have to change those whereever
those were referred.

# set up ec2 amazon linux instances

# controller
t2 micro is sufficient
inbound ports 
- 80 
- 8080
- 443
- 22

# master
- has to have atleast 2 cpus t2 medium atleast
# inbound ports
- 8080 jenkins
- TCP Inbound 6443* 
- Kubernetes API server All TCP Inbound 2379-2380 
- etcd server client API kube-apiserver, etcd TCP Inbound 10250
- kubelet API Self, Control plane TCP Inbound 10251 
- kube-scheduler Self TCP Inbound 10252 kube-controller-manager Self

# Worker node(s)
- has to have atleast 2 cpus t2 medium atleast
# inbound ports 
- 8090 for tomcat
- 8080 jenkins
- TCP Inbound 10250 kubelet API Self, 
- Control plane TCP Inbound 30000-32767 NodePort Services† All

# change hosts and permissions
- sudo vi /etc/hosts 
- add all the hosts master and node
-  [master] 
-  54.164.166.210 
-  [ansadmin] 
-  54.164.166.212 
-  [worker] 
-  54.164.166.211
-   [all] 
-   54.164.166.210 
-   54.164.166.211

( change to ip address of different servers, ip r to get the address)

- sudo vi /etc/ssh/sshd_config ( if you want to use password) 
- change #PermitRootLogin prohibit-password to Permitrootlogin yes
-  challengeresponseauthenitication no 
-  change #passwordauthentication no to passwordauthentication yes 
-  esc :wq
_ sudo systemctl restart sshd

# from ansible controller
- sudo su - 
- adduser ansadmin 
- passwd ansadmin 
- give new password (for example welcome123)
-  sudo usermod -aG root ansadmin
-   visudo (to go into /etc/sudoers)
-    add under root ansadmin ALL=(ALL) ALL
-     su - ansadmin 
-     generate ssh keys 
-     ssh-keygen (enter for all the questions)
-      cd .ssh 
-      now you will see id_rsa.pub and id_rsa keys

# in the master
- sudo su - 
- adduser master 
- passwd master 
- give new password
-  sudo usermod -aG root master ( in case of ubuntu it is group sudo, in centos it is wheel) 
-  visudo (to go into /etc/sudoers
-  )(under root add)
-   master ALL=(ALL) ALL
-    su - master

# in the worker
- sudo su - 
- adduser worker 
- passwd worker 
- give new password 
- sudo usermod -aG root worker
- visudo (to go into /etc/sudoers)
- (under root add)
- worker ALL=(ALL) ALL 
- su - worker

# REMEMBER THE PASSWORDS
# copy the public key to the hosts server 
- ssh-copy-id -i id_rsa.pub master@172.31.23.115 ( to copy to the master server) 
- ssh-copy-id -i id_rsa master@172.31.23.115 ( to copy private ip to the master server)
- ssh-copy-id -i id_rsa.pub worker@172.31.23.115 ( to copy the tomcat server) 
- ( to copy the files to the deploy server change it to your deploy pvt ip address )

# if you get an error on the password)
- ( go to the server
- go to root 
- passwd master 
- change the password 
- and try again

- or you can chmod 640 /etc/shadow and change passwd )

- press yes enter password welcome123 key will be added
# To check ssh connection
- ssh -i id_rsa deploy@172.31.23.115 ( to see if you can log in to the server change it your deploy nodes pvt ip address) 
- ssh -i id_rsa tomcat@172.31.23.115 ( to see if you can log in to the server change it your tomcat nodes pvt ip address) 
- yeah if you did !!

# now you can check if the authorized_keys exist in the copied servers .ssh folder here
- cd .ssh
-  ls 
-  exit (to go back to the ansible server VM)

# in the ansible controller
- install ansible
- sudo whoami 
- enter the password 
- sudo yum update 
- sudo amazon-linux-extras install epel 
- sudo yum install ansible 
- sudo vi /etc/ansible/hosts add hosts
[master] 172.31.30.227 ansible_pass=ansible123 ansible_user=master

[worker] 172.31.19.177 ansible_pass=ansible123 ansible_user=worker


# check if you can ssh to the host file of /etc/ansible/hosts
# clone the git repo 
- git clone the ansible playbook directory
- sudo yum install git
-  git init
-   git clone https://github.com/nalapatt/ansible-playbook-k8s-setup.git

- cd ansible-playbook-k8s-setup 
- vi hosts

# change the hosts
[master] 172.31.30.227 ansible_pass=ansible123 ansible_user=master
[worker] 172.31.19.177 ansible_pass=ansible123 ansible_user=worker 
esc :wq

- pwd (copy the path) 
- vi ansible.cfg 
- change and insert the path in
- inventory : pwd/hosts 
- esc :wq

- ansible master --list 
- ansible worker --list 
- ansible all --list
- (should show the respective hosts)
- 
-  ansible all -m ping 
-  (and ping if successful) 


- ll (shows all the files vi k8s-pkg.yml)
 # change yaml files
- ansible-playbook k8s-pkg.yml --syntax-check
-  ansible-playbook k8s-pkg.yml --extra-vars "ansible_sudo_pass=ansible123" ( set all three passwords to this to make life easier if it works then change it )
-   if done YEAH!!!

- go into the respective terminals and in root
- connect back to the terminals
- 
- sudo usermod -aG docker master
-  sudo usermod -aG docker worker

# go back to ansible controller
- vi k8s-master.yml 
- edit masters to master 
- change to your ip address of your master in apiserver-address=private ipaddressof master 
- ansible-playbook k8s-master.yml --syntax-check 
- check the syntax if alright then
-  ansible-playbook k8s-master.yml --extra-vars "ansible_sudo_pass=ansible123" 
-  if alright YEAAAAH!!!

# in the kube master
- sudo su -
- kubectl get nodes (should see the master) 
- if error (localhost:8080 refused connection) then do this is root

- mkdir -p $HOME/.kube 
- sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
- sudo chown $(id -u):$(id -g) $HOME/.kube/config

- y to overwrite config
# back in the ansible controller 
- vi k8s-workers.yml
-  change the hosts to master and ip address in hostsvars[] 
-  ansible-playbook k8s-workers.yml --extra-vars "ansible_sudo_pass=ansible123"
-   if this is done

# in the kube master
- kubectl get nodes should show all the nodes

# YEAH YOU are DONE creating the kubernetes cluster

# Ansible Installation

### Installation steps:

1. Install python and python-pip
   ```sh
   apt install python
   yum install python-pip
   ```
1. Install ansible using pip
    ```sh
    pip install ansible
   ansible --version
   ```                          
1. Create a user called ansadmin (on Control node and Managed host)  
   ```sh
   useradd ansadmin
   passwd ansadmin
   ```
1. Below command grant sudo access to ansadmin user. 
   ```sh
   echo "ansadmin ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
   ```
   
1. Log in as a ansadmin user on master and generate ssh key (on Control node)
   ```sh 
   sudo su - ansadmin
   ssh-keygen
   ```
1. Copy keys onto all ansible managed hosts (on Control node)
   ```sh 
   ssh-copy-id ansadmin@<target-server>
   ```

1. Ansible server used to create images and store on docker registry. Hence install docker, start docker services and add ansadmin to the docker group. 
   ```sh
   apt install docker
   
   # start docker services 
   service docker start
   service docker start 
   
   # add user to docker group 
   usermod -aG docker ansadmin

   ```
1. Create a directory /etc/ansible and create an inventory file called "hosts" add control node and managed hosts IP addresses to it. 
 
### Validation test

   
1. Run ansible command as ansadmin user it should be successful (Master)
   ```sh 
   ansible all -m ping
   ```
### Ansible to Install and Set Up Docker on Ubuntu

The first thing we need to do is obtain the Docker playbook and its dependencies from the do-community/ansible-playbooks repository. We need to clone this repository to a local folder inside the Ansible Control Node.

1. Clone Ansible Playbook
   ```sh
   git clone https://github.com/do-community/ansible-playbooks.git
   cd ansible-playbooks
   ```
 The files we’re interested in are located inside the docker_ubuntu1804 folder, which has the following structure:

1. Access the docker_ubuntu1804 directory and open the vars/default.yml and edit the yml file according to your need of number of containter, name, etc.
    ```sh
    cd docker_ubuntu1804
    nano vars/default.yml
    ```
We're now ready to run this playbook on one or more servers. We can use the -l flag to make sure that only a subset of servers, or a single server, is affected by the playbook. We can also use the -u flag to specify which user on the remote server we’re using to connect and execute the playbook commands on the remote hosts.
1. To execute the playbook only on server1, connecting as User1, you can use the following command:
   ```sh
   ansible-playbook playbook.yml -l server1 -u User1
    ```
## Integrating Kubernetes cluster with Ansible

1.Login to ansible server and copy public key onto kubernetes cluseter master account

2.Update hosts file with new group called kubernetes and add kubernetes master in that.

3.Create ansible playbooks to create deployment and services
 , in ansible server create a new yaml file for deployment and paste this
 ```sh
 ---
- name: Create pods using deployment 
  hosts: kubernetes 
  # become: true
  user: ubuntu
 
  tasks: 
  - name: create a deployment
    command: kubectl apply -f valaxy-deploy.yml
  ```
  similarly create a new file sor service playbook and paste this
  
  ```sh
  ---
- name: create service for deployment
  hosts: kubernetes
  # become: true
  user: ubuntu

  tasks:
  - name: create a service
    command: kubectl apply -f valaxy-service.yml
  ```
  4.Check for pods, deployments and services on kubernetes master

   ```sh
   kubectl get pods -o wide 
    kubectl get deploy -o wide
    kubectl get service -o wide
   ```
	
 5.Access application suing service IP
   ```sh
   wget <kubernetes-Master-IP>:31200
   ```
## Server Hardning using Ansible.
1. There is many ways to secure our server one of them is securing Kubernetes cluster by ansible-galaxy
 Install it by 
  ```sh
  ansible-galaxy collection install community.kubernetes
  ```
  To use it in a playbook, specify
  ```sh
  community.kubernetes.k8s_auth
  ```
  2. Next is setting auto upgrade to ur server for patch
     ,create a playbook by copying it
     ```sh
      - name: Install unattended-upgrades package
       apt:
      name: unattended-upgrades
      update_cache: yes

     - name: Enable periodic updates
     copy:
     src: 10periodic
     dest: /etc/apt/apt.conf.d/10periodic
     owner: root
     group: root
     mode: 0644
     ```
  3. SSH hardning playbook, in this playbook we have define rules to secure our ssh from any priviledge escalation and also to stop any unwanted services and softwares.
  ```sh
      ---

- hosts: all
  vars:
    allowed_ssh_networks:
      - 192.168.122.0/24
      - 10.10.10.0/24
    unnecessary_services:
      - postfix
      - telnet
    unnecessary_software:
      - tcpdump
      - nmap-ncat
      - wpa_supplicant
  tasks:
    - name: Perform full patching
      package:
        name: '*'
        state: latest

    - name: Add admin group
      group:
        name: admin
        state: present

    - name: Add local user
      user:
        name: admin
        group: admin
        shell: /bin/bash
        home: /home/admin
        create_home: yes
        state: present

    - name: Add SSH public key for user
      authorized_key:
        user: admin
        key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
        state: present

    - name: Add sudoer rule for local user
      copy:
        dest: /etc/sudoers.d/admin
        src: etc/sudoers.d/admin
        owner: root
        group: root
        mode: 0440
        validate: /usr/sbin/visudo -csf %s

    - name: Add hardened SSH config
      copy:
        dest: /etc/ssh/sshd_config
        src: etc/ssh/sshd_config
        owner: root
        group: root
        mode: 0600
      notify: Reload SSH

    - name: Add SSH port to internal zone
      firewalld:
        zone: internal
        service: ssh
        state: enabled
        immediate: yes
        permanent: yes

    - name: Add permitted networks to internal zone
      firewalld:
        zone: internal
        source: "{{ item }}"
        state: enabled
        immediate: yes
        permanent: yes
      with_items: "{{ allowed_ssh_networks }}"

    - name: Drop ssh from the public zone
      firewalld:
        zone: public
        service: ssh
        state: disabled
        immediate: yes
        permanent: yes

    - name: Remove undesirable packages
      package:
        name: "{{ unnecessary_software }}"
        state: absent

    - name: Stop and disable unnecessary services
      service:
        name: "{{ item }}"
        state: stopped
        enabled: no
      with_items: "{{ unnecessary_services }}"
      ignore_errors: yes


  handlers:
    - name: Reload SSH
      service:
        name: sshd
        state: reloaded

   ```
   
     
  
    
     
     
   







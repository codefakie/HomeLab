<h3>This is a breakdown and step by step guide how I built my Homelab K3s Cluster.</h3>

Equipment breakdown:
I had 3 Raspberry 4's laying around for another project I was gonna tinker on way back so I decided to use them along with a Lenonovo M710q Mini PC. Initially the Lenovo had 8GB of RAM, 500GB SSD and though only 4 Core CPU, I was planning on hyperthreading that puppy to make it a up to 8 VMs for a Proxmox VM and could vituarlize the Server to have 8 VMs. I beefed it up a bit adding 32GBs of RAM and a 1TB Solid state. After setting up a Proxmox cluster, I decided to wipe it and add it to my K3s HomeLab Cluster and a beefy worker node. 


I discovered https://rpi4cluster.com/ and yes that is the ultimate resourse so I will not go too deep into every step which can be found there. That site is a master class on setting up a setting up a homelab k3s cluster from Zero to Hero, so I will most certainly heap tons of praise on that site. This is my lil slice of the Pi (bad pun I know) documenting my Home Lab Journey and what I did to get my techy stuff humming..


Since I already had some Raspberry Pis laying around I decided to flash PI Imager to the 3 USBs to get started. 

pi imager:


Before you flash the OS onto the USB, it’s crucial to configure the node name and login details for each node.

Once the OS preparation for each node is complete, I set up the following cluster:

    control: 192.168.0.101
    k3sworker1: 192.168.1.102 
    k3sworker2: 192.168.1.103 



I. Config static IP for Pi

II. To disable swap on Linux, you can use the following command:

First, we should confirm that all our nodes are up:

#switch to root
sudo -su
#install nmap
apt install nmap
#scan local network range to see who is up
nmap -sP 192.168.0.1-254

# 1. Turn off swap temporary.
sudo swapoff -a

# 2. To turn of swap permanently we need to update the `CONF_SWAPSIZE` in `dphys-swapfile` file to `0`
sudo nano /etc/dphys-swapfile

# 3. set
  CONF_SWAPSIZE=0

# 4. select control + X and save the changes.


III. Cgroup configuration
If a FATA[0000] failed to find memory cgroup (v2) error surfaces during the installation of k3s, it is likely because the Pi OS lacks the required cgroup configuration. Below are the steps needed to resolve this issue:

# 1. Open the cmdline.txt file
sudo nano /boot/cmdline.txt

#2. Add below into THE END of the current line
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory

# 3. Save the file and reboot
sudo reboot

IV. First, prepare /etc/hosts file on the one control node you are on right now:

# Edit /etc/hosts with your favorite editor, mine looks like:
127.0.0.1 localhost

192.168.0.101 control01 control01.local

192.168.0.102 kube01 kube01.local
192.168.0.103 kube02 kube02.local
192.168.0.104 kube03 kube03.local


Make Life Easier we need to install Ansible

sudo apt install ansible

# Edit file /etc/ansible/hosts
[control]
control01  ansible_connection=local

[workers]
192.168.0.102  ansible_connection=ssh
192.168.0.103  ansible_connection=ssh
192.168.0.104  ansible_connection=ssh

[cube:children]
control
workers


# for user to make keys for ansible
cd
mkdir -p ~/.ssh
chmod 700 ~/.ssh
# Do not fill anything in next command just enter
ssh-keygen -t ed25519
# Copy keys to each node, for example:
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@kube01
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@kube02
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@kube03
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@kube04

Also I will copy ssh keys from my Debian VM to the Control Node and K3s Worker nodes
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.0.101
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.0.102
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.0.103
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.0.104


Use first Ansible command

ansible all -m ping -u user

You should get the hosts successfully pinged

Now we want to install ufw and disable root login


Install K3s server on Master node

Master / Control

In our case: control01

This is our primary node.

We are going to install the K3s version of Kubernetes, that is lightweight enough for out single board computers to handle. Use the following command to download and initialize K3s’ master node.


curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --node-taint CriticalAddonsOnly=true:NoExecute --bind-address 192.168.0.101 --disable-cloud-controller --disable local-storage

join worker nodes

on control node 
sudo cat /var/lib/rancher/k3s/server/node-token

then on worker nodes:
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.101:6443 K3S_TOKEN={token} sh -

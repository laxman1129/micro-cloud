# How to setup Micro-Cloud (k3s cluster) on raspberry pi 4

## Setting up raspberry pi

### Download raspbery pi os image

``` bash
# download a copy from official site using curl command
curl -Lo https://downloads.raspberrypi.org/raspios_lite_arm64/images/raspios_lite_arm64-2022-09-26/2022-09-22-raspios-bullseye-arm64-lite.img.xz
```
__Manual download:__  
download from the site https://www.raspberrypi.com/software/operating-systems/#raspberry-pi-os-64-bit

__Install xz utils for uncompressing .xz file__  
```bash
#install xz on fedora, refer link below for other distributions
sudo dnf -y install xz

#to extract the image file provide file loaction in <file>
unxz 2022-09-22-raspios-bullseye-arm64-lite.img.xz
# renaming to have smaller file name
mv 2022-09-22-raspios-bullseye-arm64-lite.img rpi.img
```

### Prepare SD cards

Insert the sd card in the card reader

```bash
#to see the sd card location in /dev (devices)
sudo fdisk -l    

#to unount the existing partitions
#device name could be /dev/mmcblkp0 OR /dev/sdb OR /dev/sdc etc,
umount /dev/mmcblkp01 

#to partition the cared using commandline
sudo fdisk /dev/mmcblk0  
#Follow below instructions

#press d : to delete the partitions (press 1/2 to delete all the partitions)
#press n  : to create a new partition
#press p : to create a primary partiion
#press enter: to provide the parameters asked
#press t : create the partion table type
#press b : for FAT paetrition table
#press w : to save the changes

# to format the disk partition type dos/FAT 
sudo mkfs.vfat /dev/mmcblk0 -I 
```

### Creating bootable image on SD card

```bash
# this command flashes the sdcard with the OS, provide image file name
sudo dd if=rpi.img of=/dev/mmcblk0 bs=8M status=progress  

#unmount the partitions if they are mounted
umount /dev/mmcblkp01
umount /dev/mmcblkp02 

```

- Remove SD card from the reader 
- insert in the pi 
- connect pi to power
- wait for a minute or so to initialize 
- this will create /boot folder on the raspberry pi card with some default files
- disconnect pi from power
- and insert in the reader again

```bash
#for headless install enable ssh and provide network details
cd /run/media/`whoami`/boot
#this tells the pi to run ssh server on startup
touch ssh    

#this file has the wifi details
touch wpa_supplicant.conf

#edit wpa_supplicant.conf and paste below content with your network info
nano wpa_supplicant.conf

#content to add
#eg.
# ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
# update_config=1
# country=US
#
# network={
# ssid=wifi-network-name
#  psk=wifi-password
# }
# 
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=[WIFI COUNTRY CODE]

network={
 ssid="[NETWORK SSID]"
 psk="[NETWORK PASSWORD]"
}

# edit cmdline.txt
sudo nano cmdline.txt

# append below content at the end so that k3s will run smoothly

cgroup_memory=1 cgroup_enable=memory

# sample line should look like below
# console=serial0,115200 console=tty1 root=PARTUUID=2dd69f04-02 rootfstype=ext4 fsck.repair=yes rootwait cgroup_memory=1 cgroup_enable=memory

#we can also add unused ip address in our subnet here
# sample will look like below
# console=serial0,115200 console=tty1 root=PARTUUID=2dd69f04-02 rootfstype=ext4 fsck.repair=yes rootwait cgroup_memory=1 cgroup_enable=memory ip=192.168.1.149::192.168.1.1:255.255.255.0:pi-3:wlp3s0:off

#ip=<available ip address>::<gateway ip address>:<subnet mask>:<hostname>:<nic-wlp3s0-or-eth0>:<turn off auto config>


```
- Remove the SDcard from the card reader
- insert the card in the raspberry pi
- wait for the pi to boot and complete initialization might take a couple of mins

```bash
#run below commands to identify which ip addresses are allocated to the pi
arp -a

# OR 

sudo nmap -sn 192.168.1.0/24 
```

### connect to raspberry pi using ssh from laptop

```bash
#if the ip of pi is 192.168.1.146 then ssh using 
ssh pi-0@192.168.1.146
# default password is raspberry

# you can execute below command to change raspberry pi config after sshing into the pi
sudo raspi-config
# change the hostname to something like rpi-kluster-x, e.g. rpi-kluster-1, rpi-kluster-2, rpi-kluster-0 etc.
```

---
---

## Setting up k3s

### configuring iptables

```bash
# switch user (su) to root, as we'll do many root related activities now
sudo su -

# to enable iptables
sudo iptables -F

# restart the pi
reboot

# after reboot ping to check if the pi is up
ping 192.168.1.146

# switch to root again
sudo su -

```

__setup other raspberry pi nodes also like this so that we can build the cluster__

### master node setup

```bash
# to download k3s, K3S_KUBECONFIG_MODE="644" for rancher
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -

# check if k3s is installed
kubectl get nodes

# to get the token while installing other nodes
cat /var/lib/rancher/k3s/server/node-token

```

### worker node setup

```bash

#login to other pi

# to install k3s as worker node non node 1, need to pass token generated in last step
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.146:6443 K3S_TOKEN=K103004a081298c30c4db4dc0ce4f194ceb0a134ce8f83e9d882971684f47cce60d::server:289e17a06cbc2d674a3e4bf8405e22ad K3S_NODE_NAME="pi-1" sh -

# to install k3s as worker node non node 2, need to pass token generated in last step
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.146:6443 K3S_TOKEN=K103004a081298c30c4db4dc0ce4f194ceb0a134ce8f83e9d882971684f47cce60d::server:289e17a06cbc2d674a3e4bf8405e22ad K3S_NODE_NAME="pi-2" sh -

# to uninstall server
sudo su -
/usr/local/bin/k3s-uninstall.sh

# to uninstall server
sudo su -
/usr/local/bin/k3s-agent-uninstall.sh
#

```

- only master node has kubectl installed on it

```bash
# run kubectl get nodes 
kubectl get nodes
kubectl get nodes -o wide

```

### install rancher (optional :- it provides GUI for k3s cluster)
```bash
#new config directory
mkdir /etc/rancher
#location of config file
mkdir /etc/rancher/rke2
#config file
cd /etc/rancher/rke2
nano config.yaml

# add below content, without has and space after that
# token can be any value
# tls-san is the current ip address. its fixed
#
# token: its lovely weather we are having I hope the weather continues
# tls-san: 
#   - 192.168.1.146


curl -sfL https://get.rancher.io | sh -

# check youtube vid for more details
```

### installing nginx servers on the cluster

- created the kubernetes deployment yaml using vscode kubernetes deployment plugin


```bash
# create deployment file
nano k3s-nginx.yaml
# paste content from the file
# to apply deployments
kubectl apply -f k3s-nginx.yaml

# to check status
kubectl get pods -o wide

# to delete all deployments 
kubectl delete deployment --all

```

```bash
```


---
---

## Reference

https://mirailabs.io/blog/building-a-microcloud/  
https://www.youtube.com/watch?v=X9fSMGkjtug   
https://docs.k3s.io/quick-start  
https://computingforgeeks.com/how-to-extract-xz-files-on-linux/  
https://linuxhint.com/how-to-use-wpa-supplicant/  
https://www.systutorials.com/docs/linux/man/5-wpa_supplicant.conf/   
https://www.ithands-on.com/2020/12/linux-101-overview-of-cgroups-control.html  
https://www.linode.com/docs/guides/what-is-iptables/  





https://programming.vip/docs/61dfc8fd787e6.html  




# **Create k3s cluster on OrangePi 5 plus devices with ansible** 

Alpha version of Readme.md expains semi-automatic installation of k3s cluster.
Semi-automatic because there are no playbooks for download, check image hash
and dd it to the device. Those actions are described in "Manual actions".
Readme contains some explanations and ansible cli commands that does initial
config and automated k3s config with ansible. Docuememnt suppose that current
management OS is macOS (i.e. ansible is installed on macOS). BSD/UNIX/Linux
part is similar and documenatation will be updated in the future.

## Manual actions
## Copy to sdcard (WARNING: doublecheck disk device name !!! )

```bash
# on osX check out the disk name with diskutil list command 
# on linux/bsd/unix you can use lsblk or parted or gsdisk or other like cdisk
# fdisk etc.

# create bootable sd card
sudo dd if=Armbian_23.11.1_Orangepi5-plus_bookworm_edge_6.7.0-rc1_minimal.img \
    of=/dev/disk2 bs=1m status=progress
```

### Insert and Boot with SD card
![First login](./images/01_fist_login_script2.png)
### List devices lsblk
```bash
lsblk
```
![Check devices ](./images/02_check_devices.png)

### On the device OrangePI 5 Plus DD from sd to nvme; Extend partition 2 (nvme0n1p2)
```bash
# WARNING: Ensure device name!!!
# root
sudo su -
# NOTE: option bs=1M is different from osX one (bs=1m) if you use it. 
dd if=/dev/mmcblk1 of=/dev/nvme0n1 bs=1M status=progress 

# or do it via ssh with xzcat like:
xzcat Armbian_23.11.1_Orangepi5-plus_jammy_edge_6.7.0-rc1.img.xz | ssh root@192.168.1.X 'dd of=/dev/nvme0n1 bs=1M status=progress'

```
![dd and resize](./images/03_dd_and_resizefs.png)
### Check and resize fs 
![check and resize fs](./images/04_resizefs.png)
```bash
or 

parted /dev/nvme0n1
print
resizepart 2 100%

e2fsck -f /dev/nvme0n1p2
resize2fs /dev/nvme0n1p2
```
### Check out that UUIDs of sdcard and nvme are the same.
```bash
blkid
```
![list and generate uuid ](./images/05_generate_new_uuid_for_sdp1.png)
### Change UUID 
```bash
tune2fs -O metadata_csum_seed -U random /dev/mmcblk1p2
e2label /dev/nvme0n1p1 bootfs
```
After changing UUIDs you must chage it in /etc/fstab accordingly. 
### Edit /etc/fstab in /dev/nvme0n1p1 (i.e. "/")
```bash
mkdir -p /tmp/ssd
mount /dev/nvme0n1p2 /tmp/ssd

# and correct UUIDs
vim /mnt/ssd/etc/fstab
```
![change etc fstab](./images/06_change_etc_fsta_uuid.png)

### TODO: recompile with va 48 bit for cilium && Disable Swap on armbian
https://charmingwebdesign.com/raspberry-pi-kubernetes-cluster-with-cilium-cni/
```bash
# TODO make ansible role
# install prerequisities 
sudo apt-get install -y git make gcc libssl-dev bc flex bison \
     libncurses5-dev libncursesw5-dev libelf-dev \
     build-essential fakeroot libaudit-dev unzip

# Linux sources 
git clone --depth 1 https://github.com/armbian/build.git armbian
cd armbian

# Configure / compile armbian kernel
./compile.sh build BOARD=orangepi5-plus BRANCH=edge BUILD_DESKTOP=no BUILD_MINIMAL=yes KERNEL_CONFIGURE=yes RELEASE=bookworm
Set  Kernel Features → Memory model and change to 48-bit Virtual Address Space

# debug 
curl -sLO https://raw.githubusercontent.com/cilium/cilium/main/contrib/k8s/k8s-cilium-exec.sh
chmod +x ./k8s-cilium-exec.sh

# install helm on master
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm


API_SERVER_IP=<YOUR API SERVER IP>
# Kubeadm default is 6443
API_SERVER_PORT=6443
helm install cilium cilium/cilium –version 1.12.1 \
–namespace kube-system \
–set kubeProxyReplacement=strict \
–set k8sServiceHost=${API_SERVER_IP} \
–set k8sServicePort=${API_SERVER_PORT}


# disable swap
systemctl mask dev-zram1.swap
vim /etc/default/armbian-zram-config
# A few lines down the file, uncomment the line that says SWAP=false:
reboot
free -h
```

### Check and Reboot 
![check and reboot](./images/07_check_fstabs_and_reboot.png)


# Configure DHCP for static records or just leave it dynamic
After installing all devices Put it in the netwrok to get IP via DHCP (you can
do it with static records in it) and get ./ansible/inventory/hosts.ini update
with the correct IPs.   

# Running Ansible Playbooks for basic cobnfig
### Installing ansible requirements
```ansible
ansible-galaxy collection install -r collections/requirements.yml
```

### Copy pub keys
Role "copy-ssh-pub-key" is not working as expected!!! it's commented out (you
can use Copy ssh key from old playbooks blueprint)

```ansible
# Adding keys to cluster (expect to have key in ~/.ssh/id_rsa.pub)
# NOTE: will ask you for user password two times (TODO: fix this)
ansible-playbook -i inventory/hosts.ini k3s-ssh-copy-id.yml -e local_user=$USER -k

# NOTE: For removing use (suppose that pub keys are ~/.ssh/*.pub )
ansible-playbook -i inventory/hosts.ini k3s-ssh-remove-ssh-pub.yml -k
```
### Bootstrap
```ansible
ansible-playbook -i inventory/hosts.ini k3s-bootstrap.yml -K
```
### Installation - runs the folloing tasks
```text
On mater node: 
 Install K3s Master node
 Install k9s on master node
 Install Helm 
 Install Cilium with Helm
 Install cilium-cli
On nodes:
 Ensure the /etc/rancher/k3s directory exists
 Deploy the modified k3s.yaml to all nodes
 Install k3s on worker nodes
```
```ansible
ansible-playbook -i inventory/hosts.ini k3s-install.yml -K --private-key=~/.ssh/id_rsa -vvvv
# ansible-playbook -i inventory/hosts.ini k3s-install.yml -K --private-key=~/.ssh/id_rsa --limit="01.master.k3s"
```

## Old playbooks
### Copy ssh key 
That step supposes that you have already generated ssh key (~/.ssh/id_rsa*). I'll execute this one only for one of the devices (--limit=<ip_address>)
```ansible
ansible-playbook -i inventory/hosts.ini playbooks/00_copy_ssh_pub_key.yml  -k --limit="02.worker.k3s"
```

If you notice an error about sshpass just install it:
```bash
brew install hudochenkov/sshpass/sshpass
```
### Setup hostnames
Edit playbook mac addresses in order to specify hostnme for each device

```ansible
ansible-playbook -i ansible/inventory/hosts.ini \
    ./ansible/playbooks/00_set_hostname_by_mac_map.yml \
     --private-key=~/.ssh/id_rsa \
     -K \
     --limit="192.168.1.2"
```
### Update/upgrade
```ansible
ansible-playbook -i ansible/inventory/hosts.ini \
    ./ansible/playbooks/01_update_upgrade.yml \
     --private-key=~/.ssh/id_rsa \
     -K \
     --limit="k3s-master-01"
```

### Config SSH server and IPv6 ;))
```ansible
ansible-playbook -i ansible/inventory/hosts.ini \
    ./ansible/playbooks/02_sshd_config_hardened.yml \
     --private-key=~/.ssh/id_rsa \
     -K \
     --limit="k3s-master-01"
```
```ansible
ansible-playbook -i ansible/inventory/hosts.ini \
    ./ansible/playbooks/03_disable_ipv6.yml \
     --private-key=~/.ssh/id_rsa \
     -K \
     --limit="k3s-master-01"
```
```ansible
ansible-playbook -i ansible/inventory/hosts.ini \
    ./ansible/playbooks/04_disable_ipv6_nmcli.yml \
     --private-key=~/.ssh/id_rsa \
     -K \
     --limit="k3s-master-01"
```
### Print Load and reboot
```ansible
ansible-playbook -i ansible/inventory/hosts.ini \
    ./ansible/playbooks/05_load_status.yml \
     --private-key=~/.ssh/id_rsa \
     -K \
     --limit="k3s-master-01"
```
```ansible
ansible-playbook -i ansible/inventory/hosts.ini \
    ./ansible/playbooks/06_reboot.yml \
     --private-key=~/.ssh/id_rsa \
     -K \
     --limit="k3s-master-01"
```

## TODOs
Place for roadmap ideas/

### Download armbian and check sha (localhost is osx TODO: for other osses) 
```ansible
# Executed in the local macOS to download and check image
ansible-playbook ./ansible/playbooks/TODO_001_download_verify_image.yml
```

### Write image to sdcard
This to be achiavable you need to set IMAGE_PATH and DISK_DEVICE.
WARNING: Make sure that disk is not wrong one!!
```ansible
# Executed in the local macOS for prepairng SD card 
ansible-playbook ./ansible/playbooks/TODO_002_resize_nvme0n1p2.yml
```

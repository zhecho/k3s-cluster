# **Create k3s cluster on OrangePi 5 plus devices with ansible** 


## Manual Part
### Installing Armibian




## Running Ansible

```ansible
# Rebooting only k3s-worker-01 node
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/10_reboot.yml --private-key=~/.ssh/id_rsa -K --limit="k3s-worker-01"

```

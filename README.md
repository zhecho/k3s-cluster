# **Create k3s cluster on OrangePi 5 plus devices with ansible**

Older Docs before tag v1.0.3

```
# Configure DHCP for static records or just leave it dynamic
After installing all devices Put it in the netwrok to get IP via DHCP (you can
do it with static records in it) and get ./ansible/inventory/hosts.ini update
with the correct IPs.

# Running Ansible Playbooks for basic config
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
# Note: Before running bootstrap role you should change "mac_to_hostname" or just:
# comment out role
#     - role: roles/set-hostname-by-mac-map
# located in k3s-bootstrap.yml file

ansible-playbook -i inventory/hosts.ini k3s-bootstrap.yml -K
```

### SSH config
```ansible
ansible-playbook -i inventory/hosts.ini k3s-sshd-hardened.yml -K
```

### Installation - runs the folloing tasks
```text
On mater nodes:
 Install K3s Master node
 Install k9s on master node
 Install Helm
 Install Cilium with Helm
 Install cilium-cli
On nodes:
 Ensure the /etc/rancher/k3s directory exists
 Modify and deploy the modified k3s.yaml to all nodes
 Install k3s on worker nodes
```

```ansible
# NOTE: if you reinstall node - use the below command with limit option to only
master and reinstalled node that way you don't have to wait other nodes
   ex:  --limit="02.worker.k3s","01.master.k3s"

TODO: Ansible to be chnged for getting k3s token from master on reinstall worker nodes..
```

```ansible
ansible-playbook -i inventory/hosts.ini k3s-install.yml -K --private-key=~/.ssh/id_rsa -vvvv

# Sometimes reload k3s-afgent is needed

```

# Configure cilium to use L2 annoncements for local Lan

## Manifests

## Reconfig cilium to use L2 Anoncement for local Lan
This step is done via ansible helm installation of cilium
check out ./ansible/roles/k3s-master-install/tasks/install_cilium_with_helm.yml

## RBAC needs to be in place in order cilium to have access to leases
```bash
# apply role, rolebinding to cilium service account that gives cilium accecss to "leases" resources
kubectl apply -f ./k8s-manifests/cilium-rbac-for-l2-annoncement.yml
```

## Deploy Cilium lb pools
```bash
kubectl apply -f k3s-cluster/mainfests/lb-red-pool.yml
kubectl apply -f k3s-cluster/mainfests/lb-blue-pool.yml
```

## Inastall helm charts (cert-manger, open-webui, argo .. etc.)
TODOs
### Inastall argo
```text
 - Config argo to look after git repo
 - Inastall cert-manager - Let's encrypt with cert-manager
 - Install prometheus operator with helm
 - export snmp_exporter for network devices
 -
```

### Inastall open-webui
```bash
# clone open-webui repo in ./k8s-manifests/
cd k8s-manifests/ && git clone https://github.com/zhecho/open-webui.git
cd open-webui/kubernetes/helm/

# Cilium L2 annoncement mode
# Requirements:
#   - there should be ip pool that is assigned to the service - this is
# achieved by setting label matching above pools (guarantied with color: blue
# tag)
#
#   - service should have:
#       spec.loadBalancerClass: io.cilium/l2-announcer
#       spec.type: LoadBalancer


# Test what should be deployed with helm template cli command:

helm template ollama ./\
    --set webui.service.labels.color="blue"\
    --set webui.service.type="loadbalancer"\
    --set webui.service.loadbalancerclass="io.cilium/l2-announcer"


# install it
kubectl create namespace ollama
helm install ollama -n ollama ./\
    --set webui.service.labels.color="blue"\
    --set webui.service.type="LoadBalancer"\
    --set webui.service.loadBalancerClass="io.cilium/l2-announcer"


```
## Accessing cluster with ssh && k9s
```bash
# Login
ssh $USER@<k3s_master_ip> && sudo su -
# Export env in order k9s to know config file
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
# Run k9s
k9s
```





















## TODOs
Place for roadmap ideas/

# Upgrade Notes

## Pre-flight

```bash
source /Users/zen0/prj/zgc/k3s-cluster/.env_sudo_pass
```

Credentials: master-01 (.6), master-02 (.7), worker-01 (.8)

Check current state across all nodes:

```bash
for node in 192.168.112.6 192.168.112.7 192.168.112.8; do
  echo "=== $node ==="
  ssh -o ConnectTimeout=10 $node "
    echo '$ARMBIAN_SUDO' | sudo -S -k grep PRETTY /etc/os-release
    uname -r
    echo '$ARMBIAN_SUDO' | sudo -S -k df -h / | grep -v Filesystem
  "
done
```

---

## Armbian OS Upgrade

Upgrade one node at a time: **worker → secondary master → primary master**

### Per-node steps (replace HOST for each node)

```bash
HOST=192.168.112.8  # worker-01 first, then .7 (master-02), then .6 (master-01)

# 1. Drain — on master-01 (.6) run this first
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl drain <NODENAME> --ignore-daemonsets --delete-emptydir-data --force"

# 2. Stop k3s — use 'k3s' for masters, 'k3s-agent' for workers
ssh $HOST "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl stop k3s"
# For workers:
# ssh $HOST "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl stop k3s-agent"

# 3. Full OS upgrade
ssh $HOST "echo '$ARMBIAN_SUDO' | sudo -S -k apt full-upgrade -y"

# 4. Verify new Armbian version
ssh $HOST "grep PRETTY /etc/os-release && uname -r"

# 5. Reboot
ssh $HOST "echo '$ARMBIAN_SUDO' | sudo -S -k reboot"

# 6. Wait for boot (30-60s for Orange Pi 5 Plus), then verify
ssh $HOST "grep PRETTY /etc/os-release && uname -r && echo '$ARMBIAN_SUDO' | sudo -S -k systemctl is-active k3s"
# For workers:
# ssh $HOST "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl is-active k3s-agent"

# 7. Uncordon — on master-01 (.6)
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl uncordon <NODENAME>"

# 8. Verify healthy
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl get nodes -o wide"
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl get pods -A --field-selector 'status.phase!=Running,status.phase!=Succeeded'"
```

### Quick one-liners per node

```bash
# Worker-01
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl drain worker-01 --ignore-daemonsets --delete-emptydir-data --force"
ssh 192.168.112.8 "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl stop k3s-agent && echo '$ARMBIAN_SUDO' | sudo -S -k apt full-upgrade -y && echo '$ARMBIAN_SUDO' | sudo -S -k reboot"
# ... wait ~60s, then:
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl uncordon worker-01"

# Master-02
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl drain master-02 --ignore-daemonsets --delete-emptydir-data --force"
ssh 192.168.112.7 "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl stop k3s && echo '$ARMBIAN_SUDO' | sudo -S -k apt full-upgrade -y && echo '$ARMBIAN_SUDO' | sudo -S -k reboot"
# ... wait ~60s, then:
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl uncordon master-02"

# Master-01 (last — expect etcd quorum loss during reboot)
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl drain master-01 --ignore-daemonsets --delete-emptydir-data --force"
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl stop k3s && echo '$ARMBIAN_SUDO' | sudo -S -k apt full-upgrade -y && echo '$ARMBIAN_SUDO' | sudo -S -k reboot"
# ... wait ~60s, API will be down (etcd quorum), then from master-02 or after master-01 boots:
ssh 192.168.112.7 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl uncordon master-01"
```

### Notes

- `systemctl stop k3s` on **masters**, `systemctl stop k3s-agent` on **workers**
- When upgrading the **last master**, expect `ServiceUnavailable` until both masters are back (etcd needs 2/3 quorum)
- Orange Pi 5 Plus takes ~60s to boot

---

## k3s Upgrade

Current: v1.32.3+k3s | Target: upgrade one minor version at a time (1.32 → 1.33 → 1.34 → 1.35)

Order: **worker → secondary master → primary master**

### Check available versions

```bash
curl -sL https://api.github.com/repos/k3s-io/k3s/releases/latest | jq -r '.tag_name'
```

### Per-node steps

```bash
# Stop k3s, upgrade, start
ssh $HOST "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl stop k3s"
# For workers:
ssh $HOST "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl stop k3s-agent"

# Install new version
ssh $HOST "echo '$ARMBIAN_SUDO' | sudo -S -k curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.33.x+k3s1 sh -"

# systemd auto-starts the service, verify
ssh $HOST "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl is-active k3s"
# For workers:
ssh $HOST "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl is-active k3s-agent"
```

### Drain/uncordon (same as OS upgrade)

```bash
# Drain before upgrading master
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl drain <NODENAME> --ignore-daemonsets --delete-emptydir-data --force"

# Uncordon after node is back
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl uncordon <NODENAME>"
```

### Full k3s upgrade one-liners (replace VERSION)

```bash
VERSION=v1.33.x+k3s1

# Worker-01
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl drain worker-01 --ignore-daemonsets --delete-emptydir-data --force"
ssh 192.168.112.8 "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl stop k3s-agent && curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=$VERSION sudo -S -k sh - && echo '$ARMBIAN_SUDO' | sudo -S -k systemctl start k3s-agent"
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl uncordon worker-01"

# Master-02
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl drain master-02 --ignore-daemonsets --delete-emptydir-data --force"
ssh 192.168.112.7 "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl stop k3s && curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=$VERSION sudo -S -k sh - && echo '$ARMBIAN_SUDO' | sudo -S -k systemctl start k3s"
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl uncordon master-02"

# Master-01 (last — expect etcd quorum loss)
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl drain master-01 --ignore-daemonsets --delete-emptydir-data --force"
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k systemctl stop k3s && curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=$VERSION sudo -S -k sh - && echo '$ARMBIAN_SUDO' | sudo -S -k systemctl start k3s"
# Wait for API, then uncordon from another master if needed
ssh 192.168.112.7 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl uncordon master-01"
```

---

## Verification (any upgrade)

```bash
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl get nodes -o wide"
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl get pods -A --field-selector 'status.phase!=Running,status.phase!=Succeeded'"
ssh 192.168.112.6 "echo '$ARMBIAN_SUDO' | sudo -S -k k3s kubectl get componentstatuses"
```

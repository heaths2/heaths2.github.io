---
title: Kubespray 설치방법
author: G.G
date: 2024-12-30 11:56:00 +0900
categories: [Blog, Orchestration]
tags: [kubernetes, k8s, kubespray]
pin: true
---


## Kubespray 설치

```bash
echo "yes" | ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519 -C $(hostname -s)
ssh-copy-id -i /root/.ssh/id_ed25519.pub localhost
```

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```

```bash
sudo apt install -y python3-venv
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

```bash
cp -rfpv inventory/sample inventory/awx
```

<details markdown="block">
  <summary>
    코드
  </summary>

**~/kubespray/inventory/awx/inventory.ini**

```bash
# This inventory describe a HA typology with stacked etcd (== same nodes as control plane)
# and 3 worker nodes
# See https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html
# for tips on building your # inventory

# Configure 'ip' variable to bind kubernetes services on a different ip than the default iface
# We should set etcd_member_name for etcd cluster. The node that are not etcd members do not need to set the value,
# or can set the empty string value.
[kube_control_plane]
master_node01 ansible_host=192.168.0.240 ip=192.168.0.240 etcd_member_name=etcd1
master_node01 ansible_host=192.168.0.240 ip=192.168.0.240 etcd_member_name=etcd1
master_node01 ansible_host=192.168.0.240 ip=192.168.0.240 etcd_member_name=etcd1
# node1 ansible_host=95.54.0.12  # ip=10.3.0.1 etcd_member_name=etcd1
# node2 ansible_host=95.54.0.13  # ip=10.3.0.2 etcd_member_name=etcd2
# node3 ansible_host=95.54.0.14  # ip=10.3.0.3 etcd_member_name=etcd3

[etcd:children]
kube_control_plane

[kube_node]
# node4 ansible_host=95.54.0.15  # ip=10.3.0.4
# node5 ansible_host=95.54.0.16  # ip=10.3.0.5
# node6 ansible_host=95.54.0.17  # ip=10.3.0.6
```

</details>

```bash
ansible master -m ping -i inventory/awx/inventory.ini
ansible-playbook -i inventory/awx/inventory.ini -become --become-user=root cluster.yml
```

```bash
mkdir -pv $HOME/.kube
sudo cp -v /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 참조
- [Kubespray 공식 리포지토리](https://github.com/kubernetes-sigs/kubespray)

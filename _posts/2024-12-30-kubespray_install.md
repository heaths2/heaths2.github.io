---
title: Kubespray 설치방법
author: G.G
date: 2024-12-30 11:56:00 +0900
categories: [Blog, Orchestration]
tags: [kubernetes, k8s, kubespray]
---

## HA 설정

Haproxy + Keepalived

```bash
sudo apt update
sudo apt install keepalived haproxy
```

MASTER 서버 설정

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```bash
cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
  
global_defs {
    notification_email {
        idc@infra.com               # 알림 이메일을 받을 주소
    }

    notification_email_from no-reply@infra.com

    smtp_server 127.0.0.1           # SMTP 서버 주소
    smtp_connect_timeout 60         # SMTP 서버 연결 타임아웃

    router_id LVS_MAIN              # 라우터 ID (서버 식별용)
    
    vrrp_skip_check_adv_addr        # Advertisement Address 검사 비활성화
    vrrp_garp_interval 0.000001     # GARP 패킷 비활성화 (IPv4)
    vrrp_gna_interval 0.000001      # GNA 패킷 비활성화 (IPv6)
    
    script_user root
    enable_script_security          # 스크립트 보안 활성화
}

vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

# Internal VRRP 인스턴스
vrrp_instance K8s-VIP {
    state MASTER                    # MASTER/BACKUP 중 선택 (주 서버에서는 MASTER)
    interface eth0                  # VIP를 할당할 네트워크 인터페이스
    virtual_router_id 240           # VRRP 그룹 ID (범위: 1 ~255, 같은 그룹에서는 동일해야 함)
    priority 101                    # 우선순위 (숫자가 높을수록 우선)
    smtp_alert                      # VRRP 인스턴스 상태 변화 시 이메일로 알림 전송
    advert_int 1                    # VRRP 신호 간격 (초 단위)


    authentication {
        auth_type PASS
        auth_pass 1234              # VRRP 인증 비밀번호
    }

    unicast_src_ip 10.1.81.246      # 현재 노드의 소스 IP
    unicast_peer {
        10.1.81.247                 # 다른 Keepalived 인스턴스의 IP 주소
    }

    virtual_ipaddress {
        10.1.81.240/32              # VIP 설정
    }

    track_script {
        chk_haproxy                 #  스크립트를 실행하여 HAProxy 상태를 확인
    }
}
EOF
```

</details>

BACKUP 서버 설정

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```bash
cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
  
global_defs {
    notification_email {
        idc@infra.com               # 알림 이메일을 받을 주소
    }

    notification_email_from no-reply@infra.com

    smtp_server 127.0.0.1           # SMTP 서버 주소
    smtp_connect_timeout 60         # SMTP 서버 연결 타임아웃

    router_id LVS_MAIN              # 라우터 ID (서버 식별용)
    
    vrrp_skip_check_adv_addr        # Advertisement Address 검사 비활성화
    vrrp_garp_interval 0.000001     # GARP 패킷 비활성화 (IPv4)
    vrrp_gna_interval 0.000001      # GNA 패킷 비활성화 (IPv6)
    
    script_user root
    enable_script_security          # 스크립트 보안 활성화
}

vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

# Internal VRRP 인스턴스
vrrp_instance K8s-VIP {
    state BACKUP                    # MASTER/BACKUP 중 선택 (주 서버에서는 MASTER)
    interface eth0                  # VIP를 할당할 네트워크 인터페이스
    virtual_router_id 240           # VRRP 그룹 ID (범위: 1 ~255, 같은 그룹에서는 동일해야 함)
    priority 100                    # 우선순위 (숫자가 높을수록 우선)
    smtp_alert                      # VRRP 인스턴스 상태 변화 시 이메일로 알림 전송
    advert_int 1                    # VRRP 신호 간격 (초 단위)


    authentication {
        auth_type PASS
        auth_pass 1234              # VRRP 인증 비밀번호
    }

    unicast_src_ip 10.1.81.246      # 현재 노드의 소스 IP
    unicast_peer {
        10.1.81.247                 # 다른 Keepalived 인스턴스의 IP 주소
    }

    virtual_ipaddress {
        10.1.81.240/32              # VIP 설정
    }

    track_script {
        chk_haproxy                 #  스크립트를 실행하여 HAProxy 상태를 확인
    }
}
EOF
```

</details>

haproxy 설정

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```bash
cat <<'EOF' > /etc/haproxy/haproxy.cfg
#--------------------------------------------------------------------#
# Haproxy settings                                                   #
#--------------------------------------------------------------------#

#--------------------------------------------------------------------#
# Global settings                                                    #
#--------------------------------------------------------------------#
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

#--------------------------------------------------------------------#
# Default Configuration                                              #
#--------------------------------------------------------------------#
defaults
        balance source
	log	global
#	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5s
        timeout client  5m
        timeout server  30m
        timeout tunnel  24d

#--------------------------------------------------------------------#
# Frontend Configuration                                             #
#--------------------------------------------------------------------#
frontend tcp_api
        mode    tcp
        bind    *:6443
#        bind    *:6443 ssl crt /etc/haproxy/ssl/server.pem alpn h2,http/1.1
        option  tcplog

        # Access Control List
        tcp-request connection accept if { src 127.0.0.1 }
        tcp-request connection accept if { src 10.1.81.0/24 }
        tcp-request connection reject

        # Backend Access Control List
        use_backend tcp_k8s_api

#--------------------------------------------------------------------#
# BackEnd Platform Configuration                                     #
#--------------------------------------------------------------------#
backend tcp_k8s_api
        mode    tcp
        balance source
        option log-health-checks
        default-server inter 5s downinter 5s fastinter 1s rise 2 fall 3 slowstart 60s maxconn 250 maxqueue 256 weight 100

        server control-node01 10.1.81.241:6443 check
        server control-node02 10.1.81.242:6443 check
        server control-node03 10.1.81.243:6443 check

#--------------------------------------------------------------------#
# HAProxy Monitoring Configuration                                   #
#--------------------------------------------------------------------#
listen stats
	mode	http
        bind :32000
        stats enable
        stats realm Kubernetes Haproxy       # 브라우저 타이틀
        stats uri /                          # stat 를 제공할 URI
        # 접근 제한 설정
        http-request deny if !{ src 127.0.0.1 } !{ src 10.1.81.0/24 }
        # 사용자 인증 ACL 설정
        acl Auth http_auth(crdentials)
        http-request auth if !Auth

# 사용자 목록
userlist crdentials
        user k8s_mon password $5$Ku7NZ.d3iw6zGckc$hHJ0RV.rjwhKDjhpdzAaJcZeOsFphxynEfguUK9OG99
EOF
```

</details>

## Kubespray 설치

| IP           | Hostname        | Describe                 |
|:------------:|:----------------|:-------------------------|
| 10.1.81.240  | haproxy-VIP     | HAProxy VIP              |
| 10.1.81.241  | master-node01   | Kubernetes Control Plane |
| 10.1.81.242  | master-node02   | Kubernetes Control Plane |
| 10.1.81.243  | master-node03   | Kubernetes Control Plane |
| 10.1.81.244  | worker-node01   | Kubernetes Woker node    |
| 10.1.81.245  | worker-node02   | Kubernetes Woker node    |
| 10.1.81.246  | haproxy01       | HAProxy + Keepalived LB  |
| 10.1.81.247  | haproxy02       | HAProxy + Keepalived LB  |
| 10.1.81.248  | postgres01      | PostgresSQL Database     |
| 10.1.81.249  | postgres02      | PostgresSQL Database     |

```bash
echo "yes" | ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519 -C $(hostname -s)
ssh-copy-id -i /root/.ssh/id_ed25519.pub localhost
```

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```

```bash
sudo apt install python3-pip
sudo apt install -y python3-venv
python3 -m venv venv
source venv/bin/activate
python3 -m pip install -r requirements.txt
```

```bash
cp -rfpv inventory/sample inventory/awx
```

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```bash
# This inventory describe a HA typology with stacked etcd (== same nodes as control plane)
# and 3 worker nodes
# See https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html
# for tips on building your # inventory

# Configure 'ip' variable to bind kubernetes services on a different ip than the default iface
# We should set etcd_member_name for etcd cluster. The node that are not etcd members do not need to set the value,
# or can set the empty string value.
[all]
control-node01 ansible_host=10.1.81.241 ip=10.1.81.241 etcd_member_name=etcd1
control-node02 ansible_host=10.1.81.242 ip=10.1.81.242 etcd_member_name=etcd2
control-node03 ansible_host=10.1.81.243 ip=10.1.81.243 etcd_member_name=etcd3
worker-node01  ansible_host=10.1.81.244 ip=10.1.81.244
worker-node02  ansible_host=10.1.81.245 ip=10.1.81.245

[kube_control_plane]
control-node01
control-node02
control-node03

[etcd]
control-node01
control-node02
control-node03

[kube_node]
worker-node01
worker-node02

[calico-rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico-rr
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

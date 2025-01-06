---
title: Kubespray 설치방법
author: G.G
date: 2024-12-30 11:56:00 +0900
categories: [Blog, Orchestration]
tags: [kubernetes, k8s, kubespray]
---

## 개요
Kubespray는 Ansible 기반의 오픈소스 프로젝트로, Kubernetes 클러스터를 빠르고 쉽게 설치하고 관리할 수 있도록 도와줍니다. 특히 고가용성(High Availability, HA) 클러스터를 구축하는 데 적합하며, 다양한 클라우드 및 베어메탈 환경에서 동작할 수 있습니다.

## 특징
1. 고가용성(HA)
- Control Plane과 etcd 클러스터의 고가용성을 보장.
- 다중 Control Plane 노드를 통해 장애 시에도 Kubernetes API 서버가 지속적으로 동작.
- 다중 etcd 노드 구성으로 데이터 안정성과 복구 능력을 보장.

2. 확장성
- 노드 추가 및 제거를 쉽게 수행 가능.
- 다양한 네트워크 플러그인(Calico, Flannel, Cilium 등) 지원.

3. 유연성
- 다양한 운영체제(Ubuntu, CentOS, RHEL 등) 지원.
- 사용자 정의가 용이하며, 설정 파일을 통해 네트워크, 인증 등 커스터마이징 가능.

4. 자동화
- Ansible 플레이북을 기반으로 설치 및 관리 자동화.
- Kubernetes와 관련된 모든 컴포넌트(apiserver, scheduler, controller-manager, etcd 등)를 자동 설정.

## 구성요소
1. Control Plane
- Kubernetes 클러스터의 중심 컴포넌트.
- 주요 구성 요소:
  - kube-apiserver: Kubernetes API 요청을 처리.
  - kube-controller-manager: 클러스터 상태를 조정.
  - kube-scheduler: 워크로드를 적절한 노드에 스케줄링.
  - etcd: 클러스터 상태 데이터를 저장하는 분산 키-값 저장소.
- HA를 위한 구성
  - Load Balancer(Haproxy/Keepalived 등)와 VIP를 통해 Control Plane 노드 간 로드 분산.
2. etcd
- Kubernetes 클러스터 상태 데이터를 저장하는 분산 키-값 저장소.
- HA를 위해 다중 노드로 구성
  - --etcd-servers에 다수의 etcd 노드를 지정하여 장애 시에도 데이터 복구 가능.
3. Load Balancer
- Control Plane 노드로 트래픽을 분산.
- VIP(Virtual IP)를 활용하여 클라이언트가 단일 엔드포인트로 접속 가능.

4. Kubespray 구성 파일
- inventory.ini: 노드 역할 정의(Control Plane, etcd, Worker 등).
- group_vars/all.yml: 글로벌 변수 정의(Kubernetes 버전, 네트워크 설정 등).
- group_vars/k8s_cluster/addons.yml: 추가 컴포넌트(dashboard, metrics-server 등) 설정.

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
    interface enX0                  # VIP를 할당할 네트워크 인터페이스
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
    interface enX0                  # VIP를 할당할 네트워크 인터페이스
    virtual_router_id 240           # VRRP 그룹 ID (범위: 1 ~255, 같은 그룹에서는 동일해야 함)
    priority 100                    # 우선순위 (숫자가 높을수록 우선)
    smtp_alert                      # VRRP 인스턴스 상태 변화 시 이메일로 알림 전송
    advert_int 1                    # VRRP 신호 간격 (초 단위)


    authentication {
        auth_type PASS
        auth_pass 1234              # VRRP 인증 비밀번호
    }

    unicast_src_ip 10.1.81.247      # 현재 노드의 소스 IP
    unicast_peer {
        10.1.81.246                 # 다른 Keepalived 인스턴스의 IP 주소
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
frontend k8s_api_front
        mode    tcp
        bind    *:8383
#        bind    *:6443 ssl crt /etc/haproxy/ssl/server.pem alpn h2,http/1.1
        option  tcplog

        # Access Control List
        tcp-request connection accept if { src 127.0.0.1 }
        tcp-request connection accept if { src 10.1.81.0/24 }
        tcp-request connection reject

        # Backend Access Control List
        use_backend k8s_api_back

#--------------------------------------------------------------------#
# BackEnd Platform Configuration                                     #
#--------------------------------------------------------------------#
backend k8s_api_back
        mode    tcp
        balance source
        option log-health-checks
        default-server check inter 5s fastinter 1s rise 2 fall 3

        server control-node01 10.1.81.241:6443
        server control-node02 10.1.81.242:6443
        server control-node03 10.1.81.243:6443

#--------------------------------------------------------------------#
# HAProxy Monitoring Configuration                                   #
#--------------------------------------------------------------------#
listen stats
	mode	http
        bind    *:32000
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
git ls-remote --heads --tags https://github.com/kubernetes-sigs/kubespray.git
git clone -b v2.25.0 https://github.com/kubernetes-sigs/kubespray.git
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
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
control-node01 ansible_host=10.1.81.241 ip=10.1.81.241 etcd_member_name=etcd1
control-node02 ansible_host=10.1.81.242 ip=10.1.81.242 etcd_member_name=etcd2
control-node03 ansible_host=10.1.81.243 ip=10.1.81.243 etcd_member_name=etcd3
worker-node01  ansible_host=10.1.81.244 ip=10.1.81.244
worker-node02  ansible_host=10.1.81.245 ip=10.1.81.245

# ## configure a bastion host if your nodes are not directly reachable
# [bastion]
# bastion ansible_host=x.x.x.x ansible_user=some_user

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

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

</details>

```bash
ansible all -m ping -i inventory/awx/inventory.ini
ansible-playbook -i inventory/awx/inventory.ini --become --become-user=root cluster.yml
```

```bash
mkdir -pv $HOME/.kube
sudo cp -v /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
source <(kubeadm completion bash)
echo "source <(kubeadm completion bash)" >> ~/.bashrc
```

## 참조
- [Kubespray 공식 리포지토리](https://github.com/kubernetes-sigs/kubespray)
- [Kubespray 공식 문서](https://kubespray.io/)

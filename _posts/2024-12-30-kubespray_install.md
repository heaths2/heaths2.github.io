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

## 환경구성
우선 아래표를 정리하고 Xen Hypervisor를 통해서 VM을 생성 후 진행.

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

### SSH 키 생성 및 복사
1. SSH 키 생성 (ed25519 알고리즘 사용)
2. SSH 키 각 노드에 복사 (Control Plane, Worker 노드)

```bash
echo "yes" | ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519 -C $(hostname -s)
```

```bash
ssh-copy-id -i /root/.ssh/id_ed25519.pub 10.1.81.241
ssh-copy-id -i /root/.ssh/id_ed25519.pub 10.1.81.242
ssh-copy-id -i /root/.ssh/id_ed25519.pub 10.1.81.243
ssh-copy-id -i /root/.ssh/id_ed25519.pub 10.1.81.244
ssh-copy-id -i /root/.ssh/id_ed25519.pub 10.1.81.245
```

### HA 설정

#### Keepalived 설치
1. Keepalived 설치
2. `/etc/keepalived/keepalived.conf`{: filepath} MASTER/BACKUP Keepalived 설정 파일 편집
3. Keepalived 서비스 재시작

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

```bash
systemctl restart keepalived.service
```


#### Haproxy 설치
1. Haproxy 설치
2. `/etc/haproxy/haproxy.cfg`{: filepath} Haproxy 설정 파일 편집
3. Haproxy 서비스 재시작

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
1. Kubespray 클론 및 준비
2. Python 환경 설정 및 의존성 설치
3. Kubespray Inventory 복사
4. Inventory 파일 구성
  - `all.yml`{: .filepath} HAProxy로 설정된 Load Balancer의 IP와 포트 지정
  - `addons.yml`{: .filepath} 추가 애드온(대시보드, Helm, Metrics Server 등)을 활성화
  - `inventory.ini`{: .filepath} Inventory 구성
5. Kubernetes 클러스터 배포
  - 클러스터 설정이 정상적인지 확인
  - Kubernetes 클러스터를 생성
6. Kubernetes 클러스터 배포
  - 클러스터 설정이 정상적인지 확인
  - Kubernetes 클러스터를 생성
7. etcd 및 apiserver 구성
  - 인증서 재생성: VIP와 SAN을 포함하여 인증서를 재생성
  - API Server 구성 파일 수정: API Server가 모든 etcd 노드와 통신하도록 구성
  - 인증서 동기화: 다른 Control Plane 노드에 인증서 복사
  - kubelet 서비스 재시작: 변경 사항을 반영하기 위해 kubelet을 재시작
8. etcd 상태 확인
9. Kubectl 자동완성 설정

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

```yaml
---
## Directory where the binaries will be installed
bin_dir: /usr/local/bin

## The access_ip variable is used to define how other nodes should access
## the node.  This is used in flannel to allow other flannel nodes to see
## this node for example.  The access_ip is really useful AWS and Google
## environments where the nodes are accessed remotely by the "public" ip,
## but don't know about that address themselves.
# access_ip: 1.1.1.1


## External LB example config
## apiserver_loadbalancer_domain_name: "elb.some.domain"
loadbalancer_apiserver:
  address: 10.1.81.240
  port: 8383

## Internal loadbalancers for apiservers
# loadbalancer_apiserver_localhost: true
# valid options are "nginx" or "haproxy"
# loadbalancer_apiserver_type: nginx  # valid values "nginx" or "haproxy"

## Local loadbalancer should use this port
## And must be set port 6443
loadbalancer_apiserver_port: 6443

## If loadbalancer_apiserver_healthcheck_port variable defined, enables proxy liveness check for nginx.
loadbalancer_apiserver_healthcheck_port: 8081

### OTHER OPTIONAL VARIABLES

## By default, Kubespray collects nameservers on the host. It then adds the previously collected nameservers in nameserverentries.
## If true, Kubespray does not include host nameservers in nameserverentries in dns_late stage. However, It uses the nameserver to make sure cluster installed safely in dns_early stage.
## Use this option with caution, you may need to define your dns servers. Otherwise, the outbound queries such as www.google.com may fail.
# disable_host_nameservers: false

## Upstream dns servers
# upstream_dns_servers:
#   - 8.8.8.8
#   - 8.8.4.4

## There are some changes specific to the cloud providers
## for instance we need to encapsulate packets with some network plugins
## If set the possible values are either 'gce', 'aws', 'azure', 'openstack', 'vsphere', 'oci', or 'external'
## When openstack is used make sure to source in the openstack credentials
## like you would do when using openstack-client before starting the playbook.
# cloud_provider:

## When cloud_provider is set to 'external', you can set the cloud controller to deploy
## Supported cloud controllers are: 'openstack', 'vsphere', 'huaweicloud' and 'hcloud'
## When openstack or vsphere are used make sure to source in the required fields
# external_cloud_provider:

## Set these proxy values in order to update package manager and docker daemon to use proxies and custom CA for https_proxy if needed
# http_proxy: ""
# https_proxy: ""
# https_proxy_cert_file: ""

## Refer to roles/kubespray-defaults/defaults/main/main.yml before modifying no_proxy
# no_proxy: ""

## Some problems may occur when downloading files over https proxy due to ansible bug
## https://github.com/ansible/ansible/issues/32750. Set this variable to False to disable
## SSL validation of get_url module. Note that kubespray will still be performing checksum validation.
# download_validate_certs: False

## If you need exclude all cluster nodes from proxy and other resources, add other resources here.
# additional_no_proxy: ""

## If you need to disable proxying of os package repositories but are still behind an http_proxy set
## skip_http_proxy_on_os_packages to true
## This will cause kubespray not to set proxy environment in /etc/yum.conf for centos and in /etc/apt/apt.conf for debian/ubuntu
## Special information for debian/ubuntu - you have to set the no_proxy variable, then apt package will install from your source of wish
# skip_http_proxy_on_os_packages: false

## Since workers are included in the no_proxy variable by default, docker engine will be restarted on all nodes (all
## pods will restart) when adding or removing workers.  To override this behaviour by only including master nodes in the
## no_proxy variable, set below to true:
no_proxy_exclude_workers: false

## Certificate Management
## This setting determines whether certs are generated via scripts.
## Chose 'none' if you provide your own certificates.
## Option is  "script", "none"
# cert_management: script

## Set to true to allow pre-checks to fail and continue deployment
# ignore_assert_errors: false

## The read-only port for the Kubelet to serve on with no authentication/authorization. Uncomment to enable.
# kube_read_only_port: 10255

## Set true to download and cache container
# download_container: true

## Deploy container engine
# Set false if you want to deploy container engine manually.
# deploy_container_engine: true

## Red Hat Enterprise Linux subscription registration
## Add either RHEL subscription Username/Password or Organization ID/Activation Key combination
## Update RHEL subscription purpose usage, role and SLA if necessary
# rh_subscription_username: ""
# rh_subscription_password: ""
# rh_subscription_org_id: ""
# rh_subscription_activation_key: ""
# rh_subscription_usage: "Development"
# rh_subscription_role: "Red Hat Enterprise Server"
# rh_subscription_sla: "Self-Support"

## Check if access_ip responds to ping. Set false if your firewall blocks ICMP.
# ping_access_ip: true

# sysctl_file_path to add sysctl conf to
# sysctl_file_path: "/etc/sysctl.d/99-sysctl.conf"

## Variables for webhook token auth https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication
kube_webhook_token_auth: false
kube_webhook_token_auth_url_skip_tls_verify: false
# kube_webhook_token_auth_url: https://...
## base64-encoded string of the webhook's CA certificate
# kube_webhook_token_auth_ca_data: "LS0t..."

## NTP Settings
# Start the ntpd or chrony service and enable it at system boot.
ntp_enabled: false
ntp_manage_config: false
ntp_servers:
  - "0.pool.ntp.org iburst"
  - "1.pool.ntp.org iburst"
  - "2.pool.ntp.org iburst"
  - "3.pool.ntp.org iburst"

## Used to control no_log attribute
unsafe_show_logs: false

## If enabled it will allow kubespray to attempt setup even if the distribution is not supported. For unsupported distributions this can lead to unexpected failures in some cases.
allow_unsupported_distribution_setup: false
```
{: file='inventory/awx/group_vars/all/all.yml'}

</details>

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```yaml
---
# Kubernetes dashboard
# RBAC required. see docs/getting-started.md for access details.
dashboard_enabled: true

# Helm deployment
helm_enabled: true

# Registry deployment
registry_enabled: true
registry_namespace: kube-system
registry_storage_class: ""
registry_disk_size: "10Gi"

# Metrics Server deployment
metrics_server_enabled: true
metrics_server_container_port: 10250
metrics_server_kubelet_insecure_tls: true
metrics_server_metric_resolution: 15s
metrics_server_kubelet_preferred_address_types: "InternalIP,ExternalIP,Hostname"
metrics_server_host_network: false
metrics_server_replicas: 1

# Rancher Local Path Provisioner
local_path_provisioner_enabled: true
local_path_provisioner_namespace: "local-path-storage"
local_path_provisioner_storage_class: "local-path"
local_path_provisioner_reclaim_policy: Delete
local_path_provisioner_claim_root: /data/volumes/opt/local-path-provisioner/
local_path_provisioner_debug: false
local_path_provisioner_image_repo: "{{ docker_image_repo }}/rancher/local-path-provisioner"
local_path_provisioner_image_tag: "v0.0.24"
local_path_provisioner_helper_image_repo: "busybox"
local_path_provisioner_helper_image_tag: "latest"

# Local volume provisioner deployment
local_volume_provisioner_enabled: true
local_volume_provisioner_namespace: kube-system
local_volume_provisioner_nodelabels:
  - kubernetes.io/hostname
  - topology.kubernetes.io/region
  - topology.kubernetes.io/zone
local_volume_provisioner_storage_classes:
  local-storage:
    host_dir: /data/volumes
    mount_dir: /data/volumes
    volume_mode: Filesystem
    fs_type: ext4
  fast-disks:
    host_dir: /data/volumes/fast-disks
    mount_dir: /data/volumes/fast-disks
    block_cleaner_command:
      - "/scripts/shred.sh"
      - "2"
    volume_mode: Filesystem
    fs_type: ext4
local_volume_provisioner_tolerations:
  - effect: NoSchedule
    operator: Exists

# CSI Volume Snapshot Controller deployment, set this to true if your CSI is able to manage snapshots
# currently, setting cinder_csi_enabled=true would automatically enable the snapshot controller
# Longhorn is an external CSI that would also require setting this to true but it is not included in kubespray
# csi_snapshot_controller_enabled: false
# csi snapshot namespace
# snapshot_controller_namespace: kube-system

# CephFS provisioner deployment
cephfs_provisioner_enabled: false
# cephfs_provisioner_namespace: "cephfs-provisioner"
# cephfs_provisioner_cluster: ceph
# cephfs_provisioner_monitors: "172.24.0.1:6789,172.24.0.2:6789,172.24.0.3:6789"
# cephfs_provisioner_admin_id: admin
# cephfs_provisioner_secret: secret
# cephfs_provisioner_storage_class: cephfs
# cephfs_provisioner_reclaim_policy: Delete
# cephfs_provisioner_claim_root: /volumes
# cephfs_provisioner_deterministic_names: true

# RBD provisioner deployment
rbd_provisioner_enabled: false
# rbd_provisioner_namespace: rbd-provisioner
# rbd_provisioner_replicas: 2
# rbd_provisioner_monitors: "172.24.0.1:6789,172.24.0.2:6789,172.24.0.3:6789"
# rbd_provisioner_pool: kube
# rbd_provisioner_admin_id: admin
# rbd_provisioner_secret_name: ceph-secret-admin
# rbd_provisioner_secret: ceph-key-admin
# rbd_provisioner_user_id: kube
# rbd_provisioner_user_secret_name: ceph-secret-user
# rbd_provisioner_user_secret: ceph-key-user
# rbd_provisioner_user_secret_namespace: rbd-provisioner
# rbd_provisioner_fs_type: ext4
# rbd_provisioner_image_format: "2"
# rbd_provisioner_image_features: layering
# rbd_provisioner_storage_class: rbd
# rbd_provisioner_reclaim_policy: Delete

# Gateway API CRDs
gateway_api_enabled: false
# gateway_api_experimental_channel: false

# Nginx ingress controller deployment
ingress_nginx_enabled: false
# ingress_nginx_host_network: false
# ingress_nginx_service_type: LoadBalancer
# ingress_nginx_service_nodeport_http: 30080
# ingress_nginx_service_nodeport_https: 30081
ingress_publish_status_address: ""
# ingress_nginx_nodeselector:
#   kubernetes.io/os: "linux"
# ingress_nginx_tolerations:
#   - key: "node-role.kubernetes.io/control-plane"
#     operator: "Equal"
#     value: ""
#     effect: "NoSchedule"
# ingress_nginx_namespace: "ingress-nginx"
# ingress_nginx_insecure_port: 80
# ingress_nginx_secure_port: 443
# ingress_nginx_configmap:
#   map-hash-bucket-size: "128"
#   ssl-protocols: "TLSv1.2 TLSv1.3"
# ingress_nginx_configmap_tcp_services:
#   9000: "default/example-go:8080"
# ingress_nginx_configmap_udp_services:
#   53: "kube-system/coredns:53"
# ingress_nginx_extra_args:
#   - --default-ssl-certificate=default/foo-tls
# ingress_nginx_termination_grace_period_seconds: 300
# ingress_nginx_class: nginx
# ingress_nginx_without_class: true
# ingress_nginx_default: false

# ALB ingress controller deployment
ingress_alb_enabled: false
# alb_ingress_aws_region: "us-east-1"
# alb_ingress_restrict_scheme: "false"
# Enables logging on all outbound requests sent to the AWS API.
# If logging is desired, set to true.
# alb_ingress_aws_debug: "false"

# Cert manager deployment
cert_manager_enabled: false
# cert_manager_namespace: "cert-manager"
# cert_manager_tolerations:
#   - key: node-role.kubernetes.io/control-plane
#     effect: NoSchedule
# cert_manager_affinity:
#  nodeAffinity:
#    preferredDuringSchedulingIgnoredDuringExecution:
#    - weight: 100
#      preference:
#        matchExpressions:
#        - key: node-role.kubernetes.io/control-plane
#          operator: In
#          values:
#          - ""
# cert_manager_nodeselector:
#   kubernetes.io/os: "linux"

# cert_manager_trusted_internal_ca: |
#   -----BEGIN CERTIFICATE-----
#   [REPLACE with your CA certificate]
#   -----END CERTIFICATE-----
# cert_manager_leader_election_namespace: kube-system

# cert_manager_dns_policy: "ClusterFirst"
# cert_manager_dns_config:
#   nameservers:
#     - "1.1.1.1"
#     - "8.8.8.8"

# cert_manager_controller_extra_args:
#   - "--dns01-recursive-nameservers-only=true"
#   - "--dns01-recursive-nameservers=1.1.1.1:53,8.8.8.8:53"

# MetalLB deployment
metallb_enabled: false
metallb_speaker_enabled: "{{ metallb_enabled }}"
metallb_namespace: "metallb-system"
# metallb_version: v0.13.9
# metallb_protocol: "layer2"
# metallb_port: "7472"
# metallb_memberlist_port: "7946"
# metallb_config:
#   speaker:
#     nodeselector:
#       kubernetes.io/os: "linux"
#     tolerations:
#       - key: "node-role.kubernetes.io/control-plane"
#         operator: "Equal"
#         value: ""
#         effect: "NoSchedule"
#   controller:
#     nodeselector:
#       kubernetes.io/os: "linux"
#     tolerations:
#       - key: "node-role.kubernetes.io/control-plane"
#         operator: "Equal"
#         value: ""
#         effect: "NoSchedule"
#   address_pools:
#     primary:
#       ip_range:
#         - 10.5.0.0/16
#       auto_assign: true
#     pool1:
#       ip_range:
#         - 10.6.0.0/16
#       auto_assign: true
#     pool2:
#       ip_range:
#         - 10.10.0.0/16
#       auto_assign: true
#   layer2:
#     - primary
#   layer3:
#     defaults:
#       peer_port: 179
#       hold_time: 120s
#     communities:
#       vpn-only: "1234:1"
#       NO_ADVERTISE: "65535:65282"
#     metallb_peers:
#         peer1:
#           peer_address: 10.6.0.1
#           peer_asn: 64512
#           my_asn: 4200000000
#           communities:
#             - vpn-only
#           address_pool:
#             - pool1
#         peer2:
#           peer_address: 10.10.0.1
#           peer_asn: 64513
#           my_asn: 4200000000
#           communities:
#             - NO_ADVERTISE
#           address_pool:
#             - pool2

argocd_enabled: false
# argocd_version: v2.11.0
# argocd_namespace: argocd
# Default password:
#   - https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli
#   ---
#   The initial password is autogenerated and stored in `argocd-initial-admin-secret` in the argocd namespace defined above.
#   Using the argocd CLI the generated password can be automatically be fetched from the current kubectl context with the command:
#   argocd admin initial-password -n argocd
#   ---
# Use the following var to set admin password
# argocd_admin_password: "password"

# The plugin manager for kubectl
krew_enabled: false
krew_root_dir: "/usr/local/krew"

# Kube VIP
kube_vip_enabled: false
# kube_vip_arp_enabled: true
# kube_vip_controlplane_enabled: true
# kube_vip_address: 192.168.56.120
# loadbalancer_apiserver:
#   address: "{{ kube_vip_address }}"
#   port: 6443
# kube_vip_interface: eth0
# kube_vip_services_enabled: false
# kube_vip_dns_mode: first
# kube_vip_cp_detect: false
# kube_vip_leasename: plndr-cp-lock
# kube_vip_enable_node_labeling: false

# Node Feature Discovery
node_feature_discovery_enabled: false
# node_feature_discovery_gc_sa_name: node-feature-discovery
# node_feature_discovery_gc_sa_create: false
# node_feature_discovery_worker_sa_name: node-feature-discovery
# node_feature_discovery_worker_sa_create: false
# node_feature_discovery_master_config:
#   extraLabelNs: ["nvidia.com"]
```
{: file='inventory/awx/group_vars/k8s_cluster/addons.yml'}

</details>

```bash
kubectl describe nodes | grep -i taint
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Taints:             <none>
Taints:             <none>
```
> `[kube_node]`에 워커 노드만 추가 하여 Pod가 생성되지 않도록 한다.
>  - 마스터 노드(Control Plane)에 일반 워크로드 Pod가 생성되지 않도록 Control plane 노드 격리
{: .prompt-info }

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```ini
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
control-node01 ansible_host=10.1.81.241 ip=10.1.81.241 etcd_member_name=etcd1
#control-node02 ansible_host=10.1.81.242 ip=10.1.81.242 etcd_member_name=etcd2
#control-node03 ansible_host=10.1.81.243 ip=10.1.81.243 etcd_member_name=etcd3
worker-node01  ansible_host=10.1.81.244 ip=10.1.81.244
worker-node02  ansible_host=10.1.81.245 ip=10.1.81.245

# ## configure a bastion host if your nodes are not directly reachable
# [bastion]
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube_control_plane]
control-node01
#control-node02
#control-node03

[etcd]
control-node01
#control-node02
#control-node03

[kube_node]
worker-node01
worker-node02

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```
{: file='inventory/awx/inventory.ini'}

</details>

```bash
ansible all -m ping -i inventory/awx/inventory.ini
ansible-playbook -i inventory/awx/inventory.ini --become --become-user=root cluster.yml
```

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```ini
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
{: file='inventory/awx/inventory.ini'}

</details>

```bash
ansible all -m ping -i inventory/awx/inventory.ini
ansible-playbook -i inventory/awx/inventory.ini --become --become-user=root cluster.yml
```

```bash
cd /etc/kubernetes/ssl/
mv -v apiserver.crt apiserver.crt.old
mv -v apiserver.key apiserver.key.old
```

```bash
kubeadm init phase certs apiserver \
  --apiserver-advertise-address=10.1.81.240 \
  --apiserver-cert-extra-sans="lb-apiserver.kubernetes.local"
```

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep "Subject Alternative Name"
```

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```yaml
    - --etcd-servers=https://10.1.81.241:2379,https://10.1.81.242:2379,https://10.1.81.243:2379
```
{: file='/etc/kubernetes/manifests/kube-apiserver.yaml'}

</details>

```bash
rsync -avhP 10.1.81.241:/etc/kubernetes/ssl/apiserver.{crt,key} /etc/kubernetes/ssl/
```

```bash
systemctl restart kubelet.service
```

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://10.1.81.241:2379,https://10.1.81.242:2379,https://10.1.81.243:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/member-control-node01.pem \
  --key=/etc/ssl/etcd/ssl/member-control-node01-key.pem \
  endpoint health
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

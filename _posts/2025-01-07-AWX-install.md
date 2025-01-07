---
title: Ansible AWX 설치방법
author: G.G
date: 2025-01-07 08:10:00 +0900
categories: [Blog, Orchestration]
tags: [ansible, awx]
---

> 관련글 :
> [ Kubespray 설치방법 (1)](https://heaths2.github.io/posts/kubespray_install/)

## 개요
Ansible AWX는 Ansible의 CLI 중심 관리에 GUI 기반의 대시보드, REST API, 작업 스케줄링 등의 기능을 추가한 플랫폼입니다. 이는 팀과 조직이 공동으로 자동화 워크플로우를 설계하고 실행하며, 작업의 상태를 모니터링할 수 있는 강력한 도구를 제공합니다.

## 특징
1. 사용자 친화적인 GUI
- CLI 없이도 작업을 실행, 모니터링, 관리 가능.
- 대시보드에서 작업 성공/실패를 직관적으로 파악 가능.

2. 중앙 집중식 관리
- 중앙에서 플레이북, 인벤토리, 자격 증명(credential) 등을 관리.
- 대규모 팀에서도 인프라 변경사항을 효율적으로 배포 가능.

3. 워크플로우 자동화
- 작업 템플릿을 연결하여 복잡한 워크플로우를 구축 가능.
- 조건부 작업 실행, 분기 설정 지원.

4. RBAC (Role-Based Access Control)
- 사용자와 팀에 따라 세분화된 접근 권한 부여.
- 특정 플레이북, 인벤토리에 대한 권한을 제한.

5. REST API 제공
- 모든 기능을 API를 통해 호출 가능.
- 외부 시스템과의 통합 용이.

6. 스케줄링
- 반복적인 작업(예: 백업, 패치 배포)을 스케줄링 가능.

7. 다양한 통합 기능
- Git, SCM(소스 코드 관리), 클라우드 플랫폼(AWS, Azure, GCP)과의 통합.
- 외부 Vault를 활용한 보안 자격 증명 관리.

## 구성요소
1. 프로젝트(Project)
- Ansible 플레이북과 역할(Role)을 저장하는 저장소.
- Git, Subversion(SVN), 파일 디렉토리와 연결 가능.

2. 인벤토리(Inventory)
- 관리 대상 시스템(호스트)의 목록.
- 그룹 단위 관리, 변수 추가 가능.

3. 자격 증명(Credential)
- Ansible 실행에 필요한 인증 정보를 저장.
- SSH 키, 클라우드 API 키, 사용자 비밀번호 등이 포함.

4. 작업 템플릿(Job Template)
- Ansible 작업의 실행 단위를 정의.
- 어떤 인벤토리와 자격 증명을 사용할지, 어떤 플레이북을 실행할지 설정.

5. 워크플로우(Workflow)
- 여러 작업 템플릿을 연결하여 복잡한 자동화 워크플로우를 설계.
- 조건부 분기, 병렬 작업 실행 지원.

6. 대시보드(Dashboard)
- 실행한 작업의 성공/실패 상태를 실시간으로 모니터링.
- 팀 전체의 활동 현황과 주요 알림 제공.

7. 사용자와 팀
- RBAC를 기반으로 작업 템플릿, 인벤토리, 프로젝트에 대한 접근 권한 관리.
- 팀 단위로 그룹화하여 관리.

## 설치

```bash
LTS_TAG=$(curl -s https://api.github.com/repos/ansible/awx-operator/releases/latest | grep tag_name | cut -d '"' -f 4)
```

```yaml
tee ./kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization 
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=$LTS_TAG
# Set the image tags to match the git version from above 
images:
  - name: quay.io/ansible/awx-operator 
    newTag: $LTS_TAG
# Specify a custom namespace in which to install AWX
namespace: awx
EOF
```
{: file='kustomization.yaml'}

```bash
kubectl apply -k ~/awx
```

```yaml
sed -i '6a\  - awx-server.yaml' ./kustomization.yaml
```
{: file='kustomization.yaml'}

```yaml
tee ./awx-server.yaml << EOF
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-server
spec:
  service_type: nodeport
  nodeport_port: 30080
EOF
```
{: file='awx-server.yaml'}

```bash
kubectl apply -k ~/awx
```

```bash
kubectl get svc awx-server-service -o yaml > awx-server-service.yaml
```

```bash
sed -i '/nodePort:/s/[0-9]\+/32000/' awx-server-service.yaml
```

```bash
kubectl apply --dry-run=client -f awx-server-service.yaml
```

```bash
kubectl apply -f awx-server-service.yaml
```

```bash
kubectl get pods -n awx -o wide --watch
awx-operator-controller-manager-79dbd7fd-9fx47   2/2     Running     0          66m   10.233.85.12   worker-node01   <none>           <none>
awx-server-migration-24.6.1-krz8m                0/1     Completed   0          82m   10.233.85.11   worker-node01   <none>           <none>
awx-server-postgres-15-0                         1/1     Running     0          85m   10.233.94.6    worker-node02   <none>           <none>
awx-server-task-9fdc7f546-hcn5h                  4/4     Running     0          84m   10.233.94.7    worker-node02   <none>           <none>
awx-server-web-5454d457b6-bfprt                  3/3     Running     0          84m   10.233.85.10   worker-node01   <none>           <none>
```

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```bash
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

frontend k8s_awx_front
        mode    http
        bind    *:80
#        bind    *:6443 ssl crt /etc/haproxy/ssl/server.pem alpn h2,http/1.1
        option  httplog
        option  forwardfor
        option  http-server-close

        # Access Control List
        http-request deny if !{ src 127.0.0.1 } !{ src 10.1.81.0/24 } !{ src 220.85.21.35/32 } !{ src 172.16.0.0/24 }

        # Backend Access Control List
        use_backend k8s_awx_back
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

backend k8s_awx_back
        mode    http
        balance roundrobin
        option log-health-checks
        default-server check inter 5s fastinter 1s rise 2 fall 3

        server control-node01 10.1.81.241:30080
        server control-node02 10.1.81.242:30080
        server control-node03 10.1.81.243:30080
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
        http-request deny if !{ src 127.0.0.1 } !{ src 10.1.81.0/24 } !{ src 220.85.21.35/32 } !{ src 172.16.0.0/24 }
        # 사용자 인증 ACL 설정
        acl Auth http_auth(crdentials)
        http-request auth if !Auth

# 사용자 목록
userlist crdentials
        user k8s_mon password $5$Ku7NZ.d3iw6zGckc$hHJ0RV.rjwhKDjhpdzAaJcZeOsFphxynEfguUK9OG99
```

</details>

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
```

```bash
systemctl restart haproxy.service
```


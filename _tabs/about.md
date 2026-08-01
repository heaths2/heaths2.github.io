---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

{% comment %}
  '소개' 단락은 직접 작성. 넣으면 좋은 것:
    - 현재 직무 / 담당 영역
    - 주로 다루는 인프라 규모 (노드 수, 사용자 수 등)
    - 관심 방향 (예: 온프렘 → 하이브리드, 자동화, AI 플랫폼)
{% endcomment %}

## 소개

인프라를 직접 구축하고 운영하며 남긴 기록입니다. 설치 절차뿐 아니라 **왜 그 선택을 했는지**를 함께 적어, 시간이 지나도 판단 근거를 되짚을 수 있게 하는 것을 목표로 합니다.

## 역량 맵


총 **57편**의 구축·운영 기록을 아래 영역으로 나눠 정리했습니다.

| 영역 | 기록 |
|---|---:|
| 인증·계정관리 | 4편 |
| 컨테이너·오케스트레이션 | 16편 |
| 네트워크·보안 | 13편 |
| 가상화·프로비저닝 | 5편 |
| DNS·IPAM | 3편 |
| 관측·로깅 | 3편 |
| AI·LLM | 1편 |
| 시스템·도구 | 7편 |
| 금융 | 5편 |


### 인증·계정관리 (4편)

OpenLDAP 자체 구축, Active Directory 운영, 리눅스–AD 통합 인증(SSSD), 비밀번호 관리까지 계정 체계 전반을 다뤘습니다.

- [OpenLDAP 설치 매뉴얼](/posts/installing-openldap/)
- [AD(Active Directory) 사용법](/posts/commands-activedirectory/)
- [Vaultwarden 설치 가이드](/posts/installing-vaultwarden/)
- [SSSD & Active Directory 연동 가이드](/posts/installing-sssd-ad/)

### 컨테이너·오케스트레이션 (16편)

kubeadm · Kubespray · RKE2 **세 가지 배포 방식**과 Ansible AWX **네 가지 설치 방식**을 직접 구축했습니다. GitOps(ArgoCD), 레지스트리(Harbor), CI(Jenkins)로 배포 파이프라인까지 연결했습니다.

📌 **[컨테이너 플랫폼 구축 기록 — kubeadm에서 RKE2까지](/posts/overview-container-platform/)** — 전체 흐름과 선택 근거를 정리한 개요

- [Kubernetes 설치방법](/posts/installing-k8s/)
- [Kubespray 설치방법](/posts/installing-kubespray/)
- [AWX Github 연동](/posts/installing-awx-github/)
- [Ansible AWX 설치방법](/posts/installing-awx/)
- [Helm을 이용한 AWX 설치](/posts/installing-awx-helm/)
- [OpenStack 설치 매뉴얼](/posts/installing-kolla-ansible/)
- [AWX & AWX-Operator 설치](/posts/installing-awx-operator/)
- [RKE2 + Rancher 설치 매뉴얼](/posts/installing-rke2/)
- [Jenkins 설치](/posts/installing-jenkins/)
- [ArgoCD 설치](/posts/installing-argocd/)
- [Harbor 설치](/posts/installing-harbor/)
- [Podman & Podman Desktop 설치 및 사용법 가이드](/posts/installing-podman/)
- [Translate a Docker Compose File to Kubernetes Resources](/posts/commands-kompose/)
- [Portainer 설치 가이드](/posts/installing-portainer/)
- [GitLab Community Edition 설치 가이드](/posts/installing-gitlab-ce/)

### 네트워크·보안 (13편)

Cisco 스위치 실장비 운영(초기화 · ACL · FHRP · 대역 제한)부터 OPNsense 방화벽, iptables/firewalld, FortiGate SSL VPN, 리버스 프록시와 인증서 발급까지.

- [Cisco Catalyst 2960 스위치 공장 초기화](/posts/commands-c2960s-factory-reset/)
- [Cisco Switch 기본 설정](/posts/commands-cisco-base-setting/)
- [Cisco Switch Bandwidth Limit 설정](/posts/commands-cisco-switch-limit/)
- [Cisco Switch 복구 X/Y/Z MODEM](/posts/commands-cisco-switch-serial-transmission/)
- [Cisco Switch FHRP 설정](/posts/commands-fhrp/)
- [FortiGate SSL VPN 설정](/posts/installing-fortigate-sslvpn/)
- [Cisco Switch 로그인 제어 설정 개요](/posts/commands-cisco-login-acl/)
- [IPtables 사용법](/posts/commands-iptables/)
- [OPNsense HAProxy 사용법](/posts/installing-opnsense-haproxy/)
- [Nginx Proxy Manager 설치](/posts/installing-nginx-proxy-manager/)
- [Certbot SSL 인증서 발급 방법](/posts/commands-certbot/)
- [Pi-hole 설치 가이드](/posts/installing-pi-hle/)
- [Firewalld 사용법](/posts/commands-firewalld/)

### 가상화·프로비저닝 (5편)

Xen · Hyper-V · KVM(virt-manager) 하이퍼바이저와 iPXE 네트워크 부팅, VM 마이그레이션.

- [XEN Hypervisor 설치방법](/posts/installing-xen-hypervisor/)
- [iPXE 설치방법](/posts/installing-ipxe/)
- [Virt-Manager 설치 매뉴얼](/posts/installing-virt-manager/)
- [Migrating VirtualBox VM to Hyper-V 방법](/posts/installing-virtualbox-to-hyperv/)
- [Hyper-V 설치 방법](/posts/installing-hyper-v/)

### DNS·IPAM (3편)

PowerDNS 권한 DNS와 웹 관리 UI, NetBox 기반 인프라 자산 관리.

- [PowerDNS & PowerDNS Admin Docker 설치](/posts/installing-powerdns/)
- [PowerDNS & PowerAdmin Web UI 설치](/posts/installing-poweradmin/)
- [NetBox Community 설치 가이드](/posts/installing-netbox/)

### 관측·로깅 (3편)

ELK 스택과 rsyslog 중앙 로그 수집, 터미널 세션 기록.

- [ELK Stack 구성](/posts/installing-elk/)
- [RSyslog 설정법](/posts/installing-rsyslog-settings/)
- [ASCiinema 사용법](/posts/commands-asciinema/)

### AI·LLM (1편)

12계층 LLM 거버넌스 플랫폼의 참조 아키텍처와 레이어별 구축 절차.

- [Enterprise LLMOps AI 거버넌스 구축 매뉴얼](/posts/installing-llmops-ai-governance/)

### 시스템·도구 (7편)

운영 중 반복적으로 쓰는 도구와 명령어 정리.

- [Hello World](/posts/commands-hello-world/)
- [Database 명령어 사용법](/posts/commands-db-base/)
- [DMI table decoder 명령어 사용법](/posts/commands-dmi/)
- [LVM(Logical Volume Management) 명령어 사용법](/posts/commands-lvm/)
- [Poetry 사용법](/posts/commands-poetry/)
- [주식 추가 매수 계산](/posts/commands-stock-average-calculator/)
- [WSL 사용법](/posts/commands-wsl/)

### 금융 (5편)

채권 · ETF · 트레이딩 관련 개인 학습 정리.

- [채권이란](/posts/finance-bond/)
- [ETF 운용사 수수료 비교](/posts/finance-fund-commission/)
- [ETF 운용사 분당금 확인](/posts/finance-fund-dividend/)
- [스윙 트레이딩 기법](/posts/finance-trading-swing/)
- [주식 호가 단위](/posts/finance-trading-price-round-figure/)

## 이 기록을 읽는 법

- **`구축` 계열** — 설치부터 검증까지 순서대로 따라갈 수 있게 쓴 글입니다.
- **`명령어` 계열** — 이미 구축된 환경에서 필요할 때 찾아보는 참조용입니다.
- 각 글의 명령어는 예시 환경 기준입니다. 도메인 · IP · 버전은 사용 환경에 맞게 바꿔야 합니다.

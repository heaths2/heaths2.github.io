---
title: NetBox Community 설치 가이드
author: G.G
date: 2026-05-10 19:30 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, IAPM, Netbox]
---

## 📘 개요 (Overview)

SSSD(System Security Services Daemon)는 리눅스 시스템에서 사용자 인증과 계정 정보를 중앙에서 통합 관리할 수 있도록 지원하는 서비스입니다. 특히 Active Directory와 연동하면, 개별 서버마다 계정을 생성하지 않고도 도메인 계정을 활용해 로그인 및 권한 관리를 일관되게 수행할 수 있습니다.

SSSD를 활용하면 LDAP 및 Kerberos 기반 인증을 통해 사용자 인증을 중앙화하고, 서버 간 계정 및 권한 정책을 표준화할 수 있습니다. 또한 캐싱 기능을 통해 네트워크가 일시적으로 단절되더라도 기존 사용자 인증을 유지할 수 있어 안정적인 운영 환경을 제공합니다.

이 가이드는 리눅스 서버에서 SSSD를 구성하고 Active Directory와 연동하여, 도메인 기반 사용자 인증 및 권한 관리를 통합하는 방법을 단계별로 설명합니다. 이를 통해 보안성과 관리 효율성을 동시에 향상시키는 것을 목표로 합니다.

## 📂 디렉토리 구조 (Tree 구조)

```bash
/etc
└── /etc/nsswitch.conf

/etc
└── /etc/krb5.conf

/etc/sssd
└── /etc/sssd/sssd.conf
```

## 🧭 SSSD 인증 플로우

```bash
[사용자 로그인]
        ↓
[PAM]
        ↓
[SSSD]  ← 핵심 엔진
        ↓
[KRB5] → 인증 (비밀번호 확인)
        ↓
[AD]   → 계정 정보
        ↓
[NSS]  → 리눅스 계정처럼 보여줌
```

## ⚙️ 설치방법

- 🧩 사전 확인 (DNS / AD 확인)

```bash
# AD LDAP 서비스 위치 확인 (SRV 레코드)
nslookup -type=SRV _ldap._tcp.infra.local

# Kerberos 인증 서버 확인
nslookup -type=SRV _kerberos._tcp.infra.local
```

{: .prompt-tip }
> 💡 **목적**
> - AD 서버 자동 탐색 가능 여부 확인
> - DNS 깨지면 → SSSD 100% 실패

- 도메인 탐색 및 Join

```bash
# 📁 작업 디렉토리 생성
APP_NAME="netbox"
BASE_DIR="/opt"
APP_DIR="${BASE_DIR}/${APP_NAME}"
mkdir -pv $APP_DIR
cd $APP_DIR

# 🔐 SELinux Context 설정
sudo semanage fcontext -a -t container_file_t "${APP_DIR}(/.*)?"
sudo restorecon -Rv "${APP_DIR}"
sudo semanage fcontext -l | grep "${APP_DIR}"
ls -ldZ "${APP_DIR}"

# DevOps 네트워크 대역 생성
podman network create \
  --driver bridge \
  --subnet 10.90.0.0/24 \
  --gateway 10.90.0.1 \
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  --ip-range 10.90.0.64/26 \
  net_devops
```

- PAM / NSS 설정 (authselect)

```bash
# SSSD + 홈디렉토리 자동 생성 + sudo 연동
git clone -b release \
  https://github.com/netbox-community/netbox-docker.git \
  "${APP_DIR}/src"
```

{: .prompt-tip }
> 💡 **목적**
> - 로그인 시 /home 자동 생성
> - sudo 권한 연동 가능

- AD 연결 검증

```bash
# AD 연결 정보 확인
sudo adcli info infra.local

# AD 사용자 조회 테스트
id bob@infra.local
```

- `/etc/sssd/sssd.conf` 설정 예시

```bash
# 사용자 이름을 bob 형태로 사용 (bob@infra.local 제거)
# sudo sed -i 's/use_fully_qualified_names =.*/use_fully_qualified_names = False/' /etc/sssd/sssd.conf

cat << EOF > /etc/sssd/sssd.conf

[sssd]
domains = infra.local
config_file_version = 2
services = nss, pam

[domain/infra.local]
default_shell = /bin/bash
krb5_store_password_if_offline = True
cache_credentials = True
krb5_realm = infra.local
realmd_tags = manages-system joined-with-adcli
id_provider = ad
fallback_homedir = /home/%u@%d
ad_domain = infra.local
use_fully_qualified_names = False
ldap_id_mapping = True
access_provider = ad
ad_access_filter = (|(memberOf=CN=sudoers_dev,OU=Users,DC=infra,DC=local)(memberOf=CN=sudoers_stg,OU=Users,DC=infra,DC=local)(memberOf=CN=sudoers_prd,OU=Users,DC=infra,DC=local))
auto_private_groups = True
ad_server = ldap.infra.local

# 성능
enumerate = False
ignore_group_members = True
entry_cache_timeout = 600
ldap_network_timeout = 5

# 접근제어
ad_gpo_access_control = disabled

# 안정성
dyndns_update = False
EOF
```

- 🧩 SSSD 재시작 및 캐시 초기화

```bash
# SSSD 중지 → 캐시 삭제 → 재시작
systemctl stop sssd
rm -f /var/lib/sss/db/* /var/lib/sss/mc/*
systemctl start sssd

# 캐시 강제 초기화
sss_cache -E
```

- 🧩 sudo 권한 확인

```bash
# 사용자 sudo 권한 확인
sudo -l -U bob@infra.local
```

## 참고 자료

- [SSSD 공식 문서](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/7/html/windows_integration_guide/ch-configuring_authentication)

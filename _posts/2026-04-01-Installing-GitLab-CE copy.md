---
title: SSSD & Active Directory 연동
author: G.G
date: 2026-04-02 09:00 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, SSSD, Active Directory, LDAP]
---

## 📘 개요 (Overview)
GitLab은 **소스 코드 관리(SCM, Source Code Management)**를 중심으로 협업 개발을 지원하는 통합 DevOps 플랫폼입니다. 개발자가 작성한 코드를 중앙 저장소에서 체계적으로 관리하고, 변경 이력을 추적하며, 여러 개발자가 동시에 안정적으로 협업할 수 있도록 도와줍니다.

또한 GitLab은 단순한 코드 저장소를 넘어, CI/CD(지속적 통합 및 배포) 기능을 통해 코드 변경 시 자동으로 빌드, 테스트, 배포까지 수행할 수 있는 환경을 제공합니다. 이를 통해 개발부터 운영까지의 전체 과정을 하나의 플랫폼에서 통합적으로 관리할 수 있습니다.

이 가이드는 GitLab Community Edition(CE)을 컨테이너 기반으로 설치하고, LDAP(Active Directory) 연동을 통해 사용자 인증을 통합하는 방법을 안내합니다.

## 📂 디렉토리 구조 (Tree 구조)

```bash
/opt/gitlab
├── docker-compose.yml     # GitLab 서비스 정의
└── .env                  # 환경 변수 (LDAP 비밀번호)

/data/gitlab
├── config                # GitLab 설정 (/etc/gitlab)
├── logs                  # 로그 (/var/log/gitlab)
├── data                  # 실제 데이터 (/var/opt/gitlab)
└── backups               # 백업 파일
```

## ⚙️ 설치방법

- 🛠️ 설치

```bash
nslookup -type=SRV _ldap._tcp.infra.local
nslookup -type=SRV _kerberos._tcp.infra.local

sudo realm list
sudo realm discover ldap.infra.local
sudo realm join -U 'infra' infra.local
sudo realm leave -U 'infra' infra.local

sudo authselect select sssd with-mkhomedir with-sudo --force
sudo authselect current

sudo adcli info infra.local
id bob@infra.local

sudo realm permit -g sudoers_dev@infra.local sudoers_stg@infra.local sudoers_prd@infra.local
sudo realm permit bob@infra.local



sudo sed -i 's/use_fully_qualified_names =.*/use_fully_qualified_names = False/' /etc/sssd/sssd.conf

sudo -l -U bob@infra.local

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

systemctl stop sssd && rm -f /var/lib/sss/db/* /var/lib/sss/mc/* && systemctl start sssd && sss_cache -E

systemctl stop sssd
rm -f /var/lib/sss/db/* /var/lib/sss/mc/*
systemctl start sssd
sss_cache -E
```

![그림_1](/assets/img/2025-09-06/그림1.png)
_Portainer 계정 생성_

## 참고 자료
- [GitLab 공식 문서](https://docs.gitlab.com/install/docker/installation)
- [GitLab-LDAP 공식 문서](https://docs.gitlab.com/administration/auth/ldap/#configure-ldap)

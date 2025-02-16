---
title: OpenLDAP 설치 매뉴얼
author: G.G
date: 2025-02-04 12:48:00 +0900
categories: [Blog, Provisioning]
tags: [provisioning, ldap, lam, ldap account manager]
---

## 개요
OpenLDAP(Open Lightweight Directory Access Protocol)는 오픈소스로 제공되는 디렉터리 서비스 소프트웨어입니다. LDAP 프로토콜을 기반으로 사용자, 그룹, 네트워크 리소스 등의 계정 및 접근 관리 기능을 제공합니다. OpenLDAP은 RFC 4510을 포함한 LDAP 관련 표준을 준수하며, 다양한 운영 체제에서 사용 가능합니다.
OpenLDAP은 기업 환경에서 중앙 인증(Authentication), 접근 제어(Access Control), 디렉터리 서비스를 제공하며, Active Directory, FreeIPA 등과 연동하여 사용할 수도 있습니다.

## 특징

| 특징 | 설명 |
|---|---|
| 오픈소스 | 자유롭게 사용 및 수정 가능 (GPL 라이선스) |
| 경량 프로토콜 | TCP/IP 기반으로 네트워크 성능 최적화 |
| 확장 가능 | 다양한 플러그인 및 백엔드 지원 |
| 표준 준수 | LDAP v3 표준 지원 |
| 보안 기능 | TLS, SASL을 통한 보안 인증 지원 |
| 다양한 운영체제 지원 | Linux, Windows, macOS 등 다수의 OS에서 동작 |
| 고가용성 | 복제(replication) 및 분산(overlay) 기능 제공 |

## 구성요소
OpenLDAP은 여러 가지 핵심 구성 요소로 이루어져 있으며, 각 구성 요소는 디렉터리 서비스의 핵심 기능을 담당합니다.

| 구성 요소 | 설명 |
|---|---|
| slapd | OpenLDAP 서버 데몬 |
| ldapsearch | LDAP 데이터를 검색하는 명령어 |
| ldapadd | 데이터를 추가하는 명령어 |
| ldapmodify | 데이터를 정_

### NSS 설정 변경 (LDAP을 NSS에 추가) 및 확인

```bash
sudo sed -i -e 's/^passwd:.*/passwd:         compat systemd ldap/' \
            -e 's/^group:.*/group:          compat systemd ldap/' \
            -e 's/^shadow:.*/shadow:         compat ldap/' /etc/nsswitch.conf

grep -wE "passwd|group|shadow|sudoers" /etc/nsswitch.conf
```

### 홈 디렉토리 자동 생성 활성화 및 기본 권한 설

```bash
sudo pam-auth-update --enable mkhomedir
sudo sed -i '/pam_mkhomedir.so/s/$/ skel=\/etc\/skel umask=077/' /etc/pam.d/common-session
```

### LDAP 서버 데이터 조회 (ldapsearch 명령어)
LDAP 서버가 정상적으로 동작하는지 확인하고, 현재 저장된 사용자 및 그룹 데이터를 조회할 수 있는 명령어.

```bash
sudo ldapsearch -x -LLL -H ldap://172.16.0.101 -b "dc=infra,dc=com"
```

## LDAP 조직 및 그룹 구조 생성

### LDAP 조직(OU) 구조 생성 및 적용용

```bash
cat <<EOF | sudo tee ~/base-groups.ldif
dn: ou=Departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: Departments

# Development OU
dn: ou=Development,ou=Departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: Development

dn: ou=Backend,ou=Development,ou=Departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: Backend

dn: ou=Frontend,ou=Development,ou=Departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: Frontend

# Infrastructure OU
dn: ou=Infrastructure,ou=Departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: Infrastructure

dn: ou=Network,ou=Infrastructure,ou=Departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: Network

dn: ou=System,ou=Infrastructure,ou=Departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: System
EOF

sudo ldapadd -x -D "cn=admin,dc=infra,dc=com" -W -f ~/base-groups.ldif
```

### LDAP 그룹 생성

```bash
cat <<EOF | sudo tee ~/groups.ldif
dn: cn=Development,ou=Departments,dc=infra,dc=com
objectClass: posixGroup
cn: Development
gidNumber: 10010

dn: cn=Backend,ou=Development,ou=Departments,dc=infra,dc=com
objectClass: posixGroup
cn: Backend
gidNumber: 10011

dn: cn=Frontend,ou=Development,ou=Departments,dc=infra,dc=com
objectClass: posixGroup
cn: Frontend
gidNumber: 10012

dn: cn=Infrastructure,ou=Departments,dc=infra,dc=com
objectClass: posixGroup
cn: Infrastructure
gidNumber: 10000

dn: cn=Network,ou=Infrastructure,ou=Departments,dc=infra,dc=com
objectClass: posixGroup
cn: Network
gidNumber: 10001

dn: cn=System,ou=Infrastructure,ou=Departments,dc=infra,dc=com
objectClass: posixGroup
cn: System
gidNumber: 10002
EOF

sudo ldapadd -x -D "cn=admin,dc=infra,dc=com" -W -f ~/groups.ldif
```

### LDAP 그룹 및 조직 구조 확인

```bash
sudo ldapsearch -x -LLL -H ldap://172.16.0.101 -b "dc=infra,dc=com" "(objectClass=organizationalUnit)"
sudo ldapsearch -x -LLL -H ldap://172.16.0.101 -b "dc=infra,dc=com" "(objectClass=posixGroup)"
```

## LDAP 사용자 계정 추가

### 조직 단위(OU) 생성 및 적용

```bash
cat <<EOF | sudo tee ~/people.ldif
dn: ou=People,dc=infra,dc=com
objectClass: organizationalUnit
ou: People
EOF

sudo ldapadd -x -D "cn=admin,dc=infra,dc=com" -W -f ~/people.ldif
```

### LDAP 사용자 계정 생성 및 적용

```bash
PW1=$(slappasswd -h {SSHA} -s "0200")
PW2=$(slappasswd -h {SSHA} -s "0201")
PW3=$(slappasswd -h {SSHA} -s "0202")
PW4=$(slappasswd -h {SSHA} -s "0205")
PW5=$(slappasswd -h {SSHA} -s "0206")

cat <<EOF | sudo tee ~/account.ldif
dn: uid=800250200,ou=People,dc=infra,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: SuperUser
sn: SuperUser
uid: 800250200
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/800250200
loginShell: /bin/bash
userPassword: $PW1

dn: uid=801250201,ou=People,dc=infra,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: NetworkUser
sn: NetworkUser
uid: 801250201
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/801250201
loginShell: /bin/bash
userPassword: $PW2

dn: uid=802250202,ou=People,dc=infra,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: SystemUser
sn: SystemUser
uid: 802250202
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/802250202
loginShell: /bin/bash
userPassword: $PW3

dn: uid=811250205,ou=People,dc=infra,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: BackendUser
sn: BackendUser
uid: 811250205
uidNumber: 10003
gidNumber: 10011
homeDirectory: /home/811250205
loginShell: /bin/bash
userPassword: $PW4

dn: uid=811250206,ou=People,dc=infra,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: FrontendUser
sn: FrontendUser
uid: 811250206
uidNumber: 10004
gidNumber: 10012
homeDirectory: /home/811250206
loginShell: /bin/bash
userPassword: $PW5
EOF

ldapadd -x -D "cn=admin,dc=infra,dc=com" -W -f ~/account.ldif
```

### 사용자 인증 테스트

```bash
sudo ldapwhoami -x -D "uid=800250200,ou=People,dc=infra,dc=com" -W
sudo ldapsearch -x -LLL -H ldap://172.16.0.101 -b "ou=People,dc=infra,dc=com" "(objectClass=posixAccount)"
sudo ldapsearch -x -LLL -H ldap://172.16.0.101 -b "dc=infra,dc=com" "uid=800250200"
```

## 참조
- [schema.OpenLDAP ](https://github.com/sudo-project/sudo/blob/main/docs/schema.OpenLDAP)


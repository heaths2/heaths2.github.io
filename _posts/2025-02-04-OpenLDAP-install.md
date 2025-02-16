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
| ldapmodify | 데이터를 수정하는 명령어 |
| slapcat | 데이터베이스 덤프를 생성하는 명령어 |
| slapadd | 백업 데이터를 복구하는 명령어 |
| LDIF | LDAP 데이터 전송을 위한 표준 형식 |

## 기능

| 기능 | 설명 |
|---|---|
| 사용자 및 그룹 관리 | 중앙 집중식 사용자 및 그룹 정보 저장 |
| 인증 및 권한 관리 | 외부 시스템과 연동하여 인증 지원 (PAM, Kerberos) |
| 계층적 데이터 저장 | 트리 구조를 사용하여 조직별 데이터 구성 |
| 복제(replication) | Master-Slave 및 Multi-Master 복제 지원 |
| 보안 강화 | TLS/SSL, SASL 인증 방식 지원 |
| 접근 제어 | ACL(Access Control List) 설정 가능 |
| 스키마 확장 | 필요에 따라 데이터 구조(스키마) 변경 가능 |

## OpenLDAP Server

### FQDN 설정(정규화된 도메인 이름)

```bash
sudo hostnamectl set-hostname ldap.infra.com

cat <<EOF | sudo tee -a /etc/hosts

# OpenLDAP FQDN 
172.16.0.101    ldap.infra.com ldap 
EOF
```

### OpenLDAP 패키지 설치

```bash
sudo apt update
sudo apt install slapd ldap-utils
```

### OpenLDAP 서버 구성

```bash
sudo dpkg-reconfigure slapd
```

### OpenLDAP 설정 및 초기 구성
ldap.conf 설정 (LDAP 서버 URI 및 Base DN 추가)

```bash
sudo sed -i -e '/^#URI[[:space:]]*ldap:\/\//a\
BASE   dc=infra,dc=com\
URI    ldap://ldap.infra.com' /etc/ldap/ldap.conf
```

### OpenLDAP 서비스 활성화

```bash
sudo systemctl enable --now slapd
```

### LDAP 서버 설정 확인 (ldapsearch)
- ldapsearch 명령어를 사용하여 LDAP 서버 설정을 조회
- Q → SASL 인증 사용
- LLL → 불필요한 정보 생략하여 출력
- Y EXTERNAL → root 권한으로 인증
- H ldapi:/// → 로컬 UNIX 소켓을 통해 LDAP 서버에 접속

```bash
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:///
```

### OpenLDAP 데이터베이스 덤프 (slapcat 명령어)
현재 LDAP 데이터베이스 내용을 출력

```bash
sudo slapcat
```

### LAM (LDAP Account Manager) 설치 및 PHP-FPM 활성화

```bash
sudo apt install ldap-account-manager php-fpm
sudo a2enconf php*-fpm
```

### LDAP DN (Distinguished Name)
LDAP DN은 LDAP 디렉터리에서 각 항목(Entry)을 고유하게 식별하는 문자열입니다. 계층적인 구조를 가지며, 개별 객체를 고유하게 구분할 수 있도록 설계됩니다.

| DN 구성 요소                   | 설명              | 예시                                       |
| -------------------------- | --------------- | ---------------------------------------- |
| `dc` (Domain Component)    | 도메인 구성 요소       | `dc=example,dc=com`                      |
| `ou` (Organizational Unit) | 조직 단위 (부서)      | `ou=HR,dc=example,dc=com`                |
| `cn` (Common Name)         | 개체 이름 (사용자, 그룹) | `cn=John Doe,ou=HR,dc=example,dc=com`    |
| `uid` (User ID)            | 사용자 ID          | `uid=johndoe,ou=Users,dc=example,dc=com` |

### LDAP 속성(Attributes)

| 속성            | 설명                                   | 예시 |
|--------------|------------------------------------|--------------------------------|
| `objectClass` | 객체 유형 지정                          | `inetOrgPerson`, `posixAccount` |
| `cn` (Common Name) | 사용자 전체 이름                       | `John Doe` |
| `sn` (Surname) | 성 (Last Name)                       | `Doe` |
| `uid` (User ID) | 사용자 ID                           | `johndoe` |
| `uidNumber` | 고유한 사용자 ID (UID)                | `1001` |
| `gidNumber` | 그룹 ID (GID)                        | `2000` |
| `homeDirectory` | 사용자 홈 디렉토리                     | `/home/johndoe` |
| `loginShell` | 로그인할 때 사용할 기본 쉘               | `/bin/bash` |
| `userPassword` | 사용자 비밀번호 (해시)                  | `{SSHA}hashedpassword` |
| `mail` | 사용자 이메일 주소                      | `johndoe@example.com` |
| `telephoneNumber` | 전화번호                              | `+82-10-1234-5678` |
| `title` | 직위 또는 직책                         | `Software Engineer` |
| `description` | 사용자 또는 그룹에 대한 설명              | `John's account for development` |
| `memberOf` | 사용자가 속한 그룹 정보                   | `cn=Developers,ou=Groups,dc=example,dc=com` |

## OpenLDAP Client

### /etc/hosts 파일에 OpenLDAP 서버 정보 추가

```bash
cat <<EOF | sudo tee -a /etc/hosts

# OpenLDAP FQDN 
172.16.0.101    ldap.infra.com ldap 
EOF
```

### LDAP NSS, PAM 및 관련 유틸리티 패키지 설치

```bash
sudo apt update
sudo apt install libnss-ldap libpam-ldap ldap-utils nslcd -y
```

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

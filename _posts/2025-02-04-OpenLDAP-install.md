---
title: OpenLDAP 설치 매뉴얼
author: G.G
date: 2025-02-04 12:48:00 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, LDAP, LAM, LDAP Account Manager]
---

## 📘 개요
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

![LDAP_1](/assets/img/2025-02-04/LDAP_1.png)
_LDAP slapd 비밀번호 설정_

![LDAP_2](/assets/img/2025-02-04/LDAP_2.png)
_LDAP slapd 비밀번호 설정 확인_

### OpenLDAP 서버 구성

```bash
sudo dpkg-reconfigure slapd
```

![LDAP_3](/assets/img/2025-02-04/LDAP_3.png)
_LDAP slapd 재구성_

![LDAP_4](/assets/img/2025-02-04/LDAP_4.png)
_LDAP slapd 도메인 설정_

![LDAP_5](/assets/img/2025-02-04/LDAP_5.png)
_LDAP slapd 조직 설정_

![LDAP_6](/assets/img/2025-02-04/LDAP_6.png)
_LDAP slapd 비밀번호 설정_

![LDAP_7](/assets/img/2025-02-04/LDAP_7.png)
_LDAP slapd 비밀번호 설정 확인_

![LDAP_8](/assets/img/2025-02-04/LDAP_8.png)
_LDAP slapd 데이터베이스 삭제 안함_

![LDAP_9](/assets/img/2025-02-04/LDAP_9.png)
_LDAP slapd 오래된 데이터베이스 이동_

### OpenLDAP 설정 및 초기 구성
ldap.conf 설정 (LDAP 서버 URI 및 Base DN 추가)

```bash
sudo sed -i -e '/^#URI[[:space:]]*ldap:\/\//a\
BASE    dc=infra,dc=com\
URI     ldap://ldap.infra.com' /etc/ldap/ldap.conf
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

```bash
sudo a2enmod proxy_fcgi setenvif
sudo a2enconf php*-fpm
sudo systemctl restart apache2
```
> PHP 소스 코드가 그대로 보이는 경우
{: .prompt-info }

![LDAP_21](/assets/img/2025-02-04/LDAP_21.png)
_LDAP LAM 구성_

![LDAP_22](/assets/img/2025-02-04/LDAP_22.png)
_LDAP LAM 서버 프로파일 수정_

![LDAP_23](/assets/img/2025-02-04/LDAP_23.png)
_LDAP LAM 서버 로그인_

![LDAP_24](/assets/img/2025-02-04/LDAP_24.png)
_LDAP LAM 서버 일반 설정_1_

![LDAP_25](/assets/img/2025-02-04/LDAP_25.png)
_LDAP LAM 서버 일반 설정_2_

![LDAP_26](/assets/img/2025-02-04/LDAP_26.png)
_LDAP LAM 서버 계정 항목 설정_1_

![LDAP_27](/assets/img/2025-02-04/LDAP_27.png)
_LDAP LAM 서버 계정 항목 설정_2_

![LDAP_28](/assets/img/2025-02-04/LDAP_28.png)
_LDAP LAM 서버 설정 저장_

![LDAP_29](/assets/img/2025-02-04/LDAP_29.png)
_LDAP LAM 서버 admin 계정 로그인_

![LDAP_30](/assets/img/2025-02-04/LDAP_30.png)
_LDAP LAM 서버 사용자 계정 및 그룹 생성_

![LDAP_30](/assets/img/2025-02-04/LDAP_30.png)
_LDAP LAM 서버 사용자 계정 및 그룹 생성_

![LDAP_31](/assets/img/2025-02-04/LDAP_31.png)
_LDAP LAM 서버 사용자 계정 및 그룹 생성 완료_

![LDAP_32](/assets/img/2025-02-04/LDAP_32.png)
_LDAP LAM 서버 사용자 계정 또는 목록 확인_

![LDAP_33](/assets/img/2025-02-04/LDAP_33.png)
_LDAP LAM 서버 사용자 계정 또는 그룹 트리구조 선택_

![LDAP_34](/assets/img/2025-02-04/LDAP_34.png)
_LDAP LAM 서버 사용자 계정 또는 그룹 트리구조_

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

## A. OpenLDAP Client

### hostname 설정

```bash
hostnamectl set-hostname ldap-client
```

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
sudo apt install libnss-ldapd libpam-ldapd ldap-utils sudo-ldap nslcd -y
```

![LDAP_10](/assets/img/2025-02-04/LDAP_10.png)
_LDAP nslcd LDAP 서버 URI 설정_

![LDAP_11](/assets/img/2025-02-04/LDAP_11.png)
_LDAP nslcd LDAP 서버 검색 base 설정_

![LDAP_35](assets/img/2025-02-04/LDAP_35.png)
_LDAP libnss-ldapd nsswitch.conf 설정_

### NSS 설정 확인

```bash
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         files systemd ldap
group:          files systemd ldap
shadow:         files ldap
gshadow:        files

hosts:          files dns ldap
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files

netgroup:       nis
sudoers:        files ldap
```
{: file='nsswitch.conf'}

### 홈 디렉토리 자동 생성 활성화 및 기본 권한 설정

```bash
sudo pam-auth-update --enable mkhomedir
sudo sed -i '/pam_mkhomedir.so/s/$/ skel=\/etc\/skel umask=077/' /etc/pam.d/common-session
```

### LDAP 서버 데이터 조회 (ldapsearch 명령어)
LDAP 서버가 정상적으로 동작하는지 확인하고, 현재 저장된 사용자 및 그룹 데이터를 조회할 수 있는 명령어.

```bash
getent passwd | grep I800250200
```

```bash
sudo ldapsearch -x -LLL -H ldap://172.16.0.101 -b "dc=infra,dc=com"
```

## B. SSSD Client

### hostname 설정

```bash
hostnamectl set-hostname sssd-client
```

### /etc/hosts 파일에 OpenLDAP 서버 정보 추가

```bash
cat <<EOF | sudo tee -a /etc/hosts

# OpenLDAP FQDN 
172.16.0.101    ldap.infra.com ldap 
EOF
```

### SSSD NSS, PAM, SUDO 및 관련 유틸리티 패키지 설치

```bash
sudo apt update
sudo apt install sssd sssd-tools ldap-utils libsss-sudo -y
```

### sssd.conf 설정 및 적용

```bash
sudo install -m 600 /dev/null /etc/sssd/sssd.conf

cat << EOF | sudo tee /etc/sssd/sssd.conf
[sssd]
config_file_version = 2
services = nss, pam, sudo
domains = LDAP

[nss]
filter_users =
filter_groups =

[pam]

[domain/LDAP]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap
sudo_provider = ldap
enumerate = true
# ignore_group_members = true
cache_credentials = false
ldap_schema = rfc2307
ldap_uri = ldap://ldap.infra.com:389
ldap_search_base = dc=infra,dc=com

ldap_user_search_base = dc=infra,dc=com
ldap_user_object_class = posixAccount
ldap_user_name = uid

ldap_group_search_base = dc=infra,dc=com
ldap_group_object_class = posixGroup
ldap_group_name = cn

ldap_id_use_start_tls = false
ldap_tls_reqcert = never
ldap_tls_cacert = /etc/ssl/certs/ca-certificates.crt

ldap_default_bind_dn = cn=admin,dc=infra,dc=com
ldap_default_authtok_type = password
ldap_default_authtok = qwer1234
access_provider = ldap
ldap_access_filter = (objectClass=posixAccount)
min_id = 1
max_id = 0
ldap_user_uuid = entryUUID
ldap_user_shell = loginShell
ldap_user_home_directory = homeDirectory
ldap_user_uid_number = uidNumber
ldap_user_gid_number = gidNumber
ldap_group_gid_number = gidNumber
ldap_group_uuid = entryUUID
ldap_group_member = memberUid

ldap_auth_disable_tls_never_use_in_production = true
use_fully_qualified_names = false
ldap_access_order = filter
debug_level=6

override_homedir = /home/%U
fallback_homedir = /home/%U
EOF

sudo systemctl enable --now sssd.service
```

### 홈 디렉토리 자동 생성 활성화 및 기본 권한 설정

```bash
sudo pam-auth-update --enable mkhomedir
sudo sed -i '/pam_mkhomedir.so/s/$/ skel=\/etc\/skel umask=077/' /etc/pam.d/common-session
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
#!/usr/bin/env bash

# LDAP 관리자 계정 정보
LDAP_ADMIN="cn=admin,dc=infra,dc=com"
LDAP_BASE="ou=People,dc=infra,dc=com"

# 사용자 정보 (USER_ID, CN, SN, UID_NUMBER, GID_NUMBER, PASSWORD)
declare -A USERS=(
    ["I800250200"]="SuperUser SuperUser 10001 10000 0200"
    ["I801250201"]="SuperUser SuperUser 10002 10000 0201"
    ["I802250202"]="SuperUser SuperUser 10003 10000 0202"
    ["I811250205"]="SuperUser SuperUser 10004 10000 0205"
    ["I811250206"]="SuperUser SuperUser 10005 10000 0206"
)

# LDIF 파일 경로
LDIF_FILE="/tmp/account.ldif"

# 기존 LDIF 파일 삭제 후 새로 생성
rm -f "$LDIF_FILE"

echo "LDAP 사용자 추가 스크립트 시작"

# 🔹 `UID` 대신 `USER_ID` 사용 (Bash 예약 변수 문제 방지)
for USER_ID in "${!USERS[@]}"; do
    IFS=' ' read -r CN SN UID_NUMBER GID_NUMBER PASSWORD <<< "${USERS[$USER_ID]}"

    # SSHA 해시된 비밀번호 생성
    HASHED_PASS=$(slappasswd -h {SSHA} -s "$PASSWORD" -n)

    # LDIF 파일에 사용자 추가
    cat <<EOF >> "$LDIF_FILE"
dn: uid=$USER_ID,$LDAP_BASE
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
shadowinactive: 90
shadowlastchange: 20045
shadowmax: 90
shadowmin: 1
shadowwarning: 7
cn: $CN
sn: $SN
uid: $USER_ID
uidNumber: $UID_NUMBER
gidNumber: $GID_NUMBER
homeDirectory: /home/$USER_ID
loginShell: /bin/bash
userPassword: $HASHED_PASS

EOF
done

# LDIF 파일이 정상적으로 생성되었는지 확인
if [[ ! -f "$LDIF_FILE" || ! -s "$LDIF_FILE" ]]; then
    echo "❌ LDIF 파일 생성 실패: $LDIF_FILE"
    exit 1
fi

echo "✅ LDIF 파일 생성 완료: $LDIF_FILE"

# LDAP 서버에 사용자 추가
ldapadd -x -D "$LDAP_ADMIN" -W -f "$LDIF_FILE"

echo "✅ LDAP 사용자 등록 완료"

ldapadd -x -D "cn=admin,dc=infra,dc=com" -W -f ~/account.ldif
```

### 사용자 인증 테스트

```bash
sudo ldapwhoami -x -D "uid=800250200,ou=People,dc=infra,dc=com" -W
sudo ldapsearch -x -LLL -H ldap://172.16.0.101 -b "ou=People,dc=infra,dc=com" "(objectClass=posixAccount)"
sudo ldapsearch -x -LLL -H ldap://172.16.0.101 -b "dc=infra,dc=com" "uid=I800250200"
```

```bash
getent passwd I800250200
I800250200:x:10001:10000:SuperUser:/home/I800250200:/bin/bash
```
> `I800250200` 계정 정보 확인
{: .prompt-info }

### nscd(Naming Service Cache Daemon) 캐시도 지우기

```bash
nscd --invalidate=passwd
nscd --invalidate=group
```

## sudo 권한

### 모듈 스키마 추가

```bash
cat <<'EOF' | sudo tee /etc/ldap/schema/sudoers.ldif
dn: cn=sudo,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: sudo
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.1 NAME 'sudoUser' DESC 'User(s) who may  run sudo' EQUALITY caseExactMatch SUBSTR caseExactSubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.2 NAME 'sudoHost' DESC 'Host(s) who may run sudo' EQUALITY caseExactIA5Match SUBSTR caseExactIA5SubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.3 NAME 'sudoCommand' DESC 'Command(s) to be executed by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.4 NAME 'sudoRunAs' DESC 'User(s) impersonated by sudo (deprecated)' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.5 NAME 'sudoOption' DESC 'Options(s) followed by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.6 NAME 'sudoRunAsUser' DESC 'User(s) impersonated by sudo' EQUALITY caseExactMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.7 NAME 'sudoRunAsGroup' DESC 'Group(s) impersonated by sudo' EQUALITY caseExactMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.8 NAME 'sudoNotBefore' DESC 'Start of time interval for which the entry is valid' EQUALITY generalizedTimeMatch ORDERING generalizedTimeOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.24 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.9 NAME 'sudoNotAfter' DESC 'End of time interval for which the entry is valid' EQUALITY generalizedTimeMatch ORDERING generalizedTimeOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.24 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.10 NAME 'sudoOrder' DESC 'an integer to order the sudoRole entries' EQUALITY integerMatch ORDERING integerOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 )
olcObjectClasses: ( 1.3.6.1.4.1.15953.9.2.1 NAME 'sudoRole' SUP top STRUCTURAL DESC 'Sudoer Entries' MUST ( cn ) MAY ( sudoUser $ sudoHost $ sudoCommand $ sudoRunAs $ sudoRunAsUser $ sudoRunAsGroup $ sudoOption $ sudoOrder $ sudoNotBefore $ sudoNotAfter $ description ) )
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/sudoers.ldif
```

### sudo 조직 단위(OU) 생성 및 적용

```bash
cat <<'EOF' | sudo tee ~/sudo_ou.ldif
dn: ou=SUDOers,dc=infra,dc=com
objectclass: organizationalUnit
ou: SUDOers
EOF

sudo ldapadd -x -D "cn=admin,dc=infra,dc=com" -W -f ~/sudo_ou.ldif
```

### sudo 권한 추가

```bash
cat <<'EOF' | sudo tee ~/sudo_role.ldif
dn: cn=sudo,ou=SUDOers,dc=infra,dc=com
cn: sudo
objectclass: sudoRole
objectclass: top
sudocommand: !/bin/rm
sudocommand: !/bin/mv
sudocommand: /bin/bash
sudohost: ALL
sudouser: I800250200
EOF

sudo ldapadd -x -D "cn=admin,dc=infra,dc=com" -W -f ~/sudo_role.ldif
```

### sudo 권한 적용

```bash
cat <<'EOF' | sudo tee -a /etc/ldap/ldap.conf

BASE            dc=infra,dc=com
URI             ldap://ldap.infra.com
SUDOERS_BASE    ou=SUDOers,dc=infra,dc=com
EOF

sudo systemctl restart nscd nslcd
```

## logging

```bash
ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config -s base | grep -i LOG

cat <<'EOF' | sudo tee ~/access_logging.ldif
dn: cn=config
replace: olcLogLevel
olcLogLevel: -1
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f ~/access_logging.ldif

ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config -s base | grep -i LOG



cat <<'EOF' | sudo tee -a /etc/rsyslog.d/50-default.conf

#
# Asynchronous Logging for the OpenLDAP system.
#
LOCAL4.*                        -/var/log/ldap/slapd.log
EOF

cat <<'EOF' | sudo tee ~/audit_logging.ldif
dn: olcOverlay=auditlog,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcAuditLogConfig
olcOverlay: auditlog
olcAuditlogFile: /tmp/auditlog.ldif
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f ~/audit_logging.ldif




cat <<'EOF' | sudo tee ~/audit_logging.ldif
dn: olcOverlay=auditlog,olcDatabase={1}mdb,cn=config
changetype: modify
objectClass: olcOverlayConfig
objectClass: olcAuditLogConfig
olcOverlay: auditlog
olcAuditlogFile: /var/log/ldap/audit/audit.ldif
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f ~/audit_logging.ldif


cat <<'EOF' | sudo tee ~/modify_audit_logging.ldif
dn: olcOverlay=auditlog,olcDatabase={1}mdb,cn=config
changetype: modify

replace: olcAuditlogFile
olcAuditlogFile: /var/log/ldap/audit/audit.ldif
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f ~/audit_logging.ldif


cat <<'EOF' | sudo tee ~/add_audit_logging.ldif
dn: olcOverlay=auditlog,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcAuditLogConfig
olcOverlay: auditlog
olcAuditlogFile: /tmp/auditlog.ldif
EOF
```

## Active Directory & SSSD

```bash
sudo dnf install -y samba-common-tools realmd oddjob oddjob-mkhomedir sssd adcli krb5-workstation sssd-tools

systemctl enable oddjobd.service --now
```

```bash
# kinit administrator
# kinit bob
# klist
realm discover -v infra.com
realm join -v -U bob infra.com
```

```bash
# hostname -f
Get-ADComputer -Filter 'Name -like "Woker0*"' | Select-Object Name,DNSHostName

Get-ADGroup SUDO | Select-Object DistinguishedName
```

```bash
cat <<EOF | sudo tee -a /etc/sssd/sssd.conf
use_fully_qualified_names = False
ldap_access_filter = (memberOf=CN=SUDO,OU=INFRA,DC=infra,DC=com)
EOF
```

```bash
sssctl domain-status infra.com
sssctl user-checks bob
id bob@infra.com
id bob
sudo -l -U bob
```

## 참조
- [schema.OpenLDAP ](https://github.com/sudo-project/sudo/blob/main/docs/schema.OpenLDAP)
- [RedHat 공식문서](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/8/html/integrating_rhel_systems_directly_with_windows_active_directory/connecting-directly-to-ad_connecting-rhel-systems-directly-to-ad-using-sssd)
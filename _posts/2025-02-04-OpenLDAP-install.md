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

## OpenLDAP 설치

### FQDN 설정(정규화된 도메인 이름)

```bash
sudo hostnamectl set-hostname ldap.infra.com

cat <<EOF | sudo tee -a /etc/hosts

# OpenLDAP FQDN 
10.1.1.100    ldap.infra.com ldap 
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

### OpenLDAP 서비스 활성화 및 상태 확인

```bash
sudo systemctl enable --now slapd
sudo systemctl status slapd
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

### 기본 그룹 설정 (LDIF 파일 생성 및 적용)
- ou=groups,dc=infra,dc=com → 그룹을 저장할 OU(조직 단위)
- cn=admins → 관리자 그룹 (gidNumber=2000)
- cn=developers → 개발자 그룹 (gidNumber=2001)
- cn=guests → 게스트 그룹 (gidNumber=2002)
- objectClass: posixGroup → POSIX 환경에서 사용 가능한 그룹으로 설정됨.

```bash
cat <<EOF | sudo tee ~/base-groups.ldif
dn: ou=groups,dc=infra,dc=com
objectClass: organizationalUnit
ou: groups

dn: cn=admins,ou=groups,dc=infra,dc=com
objectClass: posixGroup
cn: admins
gidNumber: 2000

dn: cn=developers,ou=groups,dc=infra,dc=com
objectClass: posixGroup
cn: developers
gidNumber: 2001

dn: cn=guests,ou=groups,dc=infra,dc=com
objectClass: posixGroup
cn: guests
gidNumber: 2002
EOF
```

### LDAP에 그룹 추가
위에서 생성한 base-groups.ldif 파일을 사용하여 LDAP에 그룹을 추가합니다.
- x → 단순 인증(Simple authentication) 사용
- D "cn=admin,dc=infra,dc=com" → LDAP 관리자 계정
- W → 비밀번호 입력 프롬프트 표시
- f ~/base-groups.ldif → LDIF 파일을 사용하여 그룹 추가

```bash
ldapadd -x -D "cn=admin,dc=infra,dc=com" -W -f ~/base-groups.ldif
```

### 추가된 그룹 확인

```bash
ldapsearch -x -LLL -b "ou=groups,dc=infra,dc=com"
```

```bash
sudo apt install ldap-account-manager php-fpm
```

```bash
sudo a2enconf php*-fpm
```

### LDAP 트리 구조

```bash
dc=infra,dc=com  (조직: infra)
 ├── ou=departments,dc=infra,dc=com  (부서 전체)
 │     ├── ou=개발팀,ou=departments,dc=infra,dc=com  (개발팀)
 │     │     ├── ou=백엔드,ou=개발팀,ou=departments,dc=infra,dc=com  (백엔드 팀)
 │     │     │     ├── uid=user1
 │     │     │     ├── uid=user2
 │     │     ├── ou=프론트엔드,ou=개발팀,ou=departments,dc=infra,dc=com  (프론트엔드 팀)
 │     │           ├── uid=user14
 │     │           ├── uid=user15
 │     │
 │     ├── ou=마케팅,ou=departments,dc=infra,dc=com  (마케팅)
 │     │     ├── ou=디지털광고,ou=마케팅,ou=departments,dc=infra,dc=com  (디지털 광고 팀)
 │     │     │     ├── uid=user3
 │     │     │     ├── uid=user4
 │     │     ├── ou=콘텐츠기획,ou=마케팅,ou=departments,dc=infra,dc=com  (콘텐츠 기획 팀)
 │     │           ├── uid=user5
 │     │
 │     ├── ou=영업,ou=departments,dc=infra,dc=com  (영업)
 │     │     ├── ou=국내영업,ou=영업,ou=departments,dc=infra,dc=com  (국내 영업 팀)
 │     │     │     ├── uid=user6
 │     │     │     ├── uid=user7
 │     │     ├── ou=해외영업,ou=영업,ou=departments,dc=infra,dc=com  (해외 영업 팀)
 │     │           ├── uid=user8
 │     │
 │     ├── ou=인프라,ou=departments,dc=infra,dc=com  (인프라)
 │     │     ├── ou=서버관리,ou=인프라,ou=departments,dc=infra,dc=com  (서버 관리 팀)
 │     │     │     ├── uid=user1
 │     │     │     ├── uid=user9
 │     │     ├── ou=네트워크관리,ou=인프라,ou=departments,dc=infra,dc=com  (네트워크 관리 팀)
 │     │           ├── uid=user10
 │     │
 │     ├── ou=HR,ou=departments,dc=infra,dc=com  (HR / 경영지원팀)
 │           ├── ou=채용,ou=HR,ou=departments,dc=infra,dc=com  (채용팀)
 │           │     ├── uid=user11
 │           ├── ou=인사관리,ou=HR,ou=departments,dc=infra,dc=com  (인사관리팀)
 │                 ├── uid=user12
 │                 ├── uid=user13
```

### LDAP 그룹 생성 LDIF

```bash
dn: dc=infra,dc=com
objectClass: top
objectClass: domain
dc: infra

dn: ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: departments

# 개발팀
dn: ou=개발팀,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 개발팀

dn: ou=백엔드,ou=개발팀,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 백엔드

dn: ou=프론트엔드,ou=개발팀,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 프론트엔드

# 마케팅
dn: ou=마케팅,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 마케팅

dn: ou=디지털광고,ou=마케팅,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 디지털광고

dn: ou=콘텐츠기획,ou=마케팅,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 콘텐츠기획

# 영업
dn: ou=영업,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 영업

dn: ou=국내영업,ou=영업,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 국내영업

dn: ou=해외영업,ou=영업,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 해외영업

# 인프라
dn: ou=인프라,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 인프라

dn: ou=서버관리,ou=인프라,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 서버관리

dn: ou=네트워크관리,ou=인프라,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 네트워크관리

# HR (경영지원팀)
dn: ou=HR,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: HR

dn: ou=채용,ou=HR,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 채용

dn: ou=인사관리,ou=HR,ou=departments,dc=infra,dc=com
objectClass: organizationalUnit
ou: 인사관리
```

### LDAP 그룹 추가 (groups.ldif)

```bash
dn: cn=개발팀,ou=개발팀,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 개발팀
gidNumber: 2000

dn: cn=백엔드,ou=백엔드,ou=개발팀,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 백엔드
gidNumber: 2001

dn: cn=프론트엔드,ou=프론트엔드,ou=개발팀,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 프론트엔드
gidNumber: 2002

dn: cn=마케팅,ou=마케팅,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 마케팅
gidNumber: 2003

dn: cn=디지털광고,ou=디지털광고,ou=마케팅,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 디지털광고
gidNumber: 2004

dn: cn=콘텐츠기획,ou=콘텐츠기획,ou=마케팅,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 콘텐츠기획
gidNumber: 2005

dn: cn=영업,ou=영업,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 영업
gidNumber: 2006

dn: cn=국내영업,ou=국내영업,ou=영업,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 국내영업
gidNumber: 2007

dn: cn=해외영업,ou=해외영업,ou=영업,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 해외영업
gidNumber: 2008

dn: cn=인프라,ou=인프라,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 인프라
gidNumber: 2009

dn: cn=서버관리,ou=서버관리,ou=인프라,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 서버관리
gidNumber: 2010

dn: cn=네트워크관리,ou=네트워크관리,ou=인프라,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 네트워크관리
gidNumber: 2011

dn: cn=HR,ou=HR,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: HR
gidNumber: 2012

dn: cn=채용,ou=채용,ou=HR,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 채용
gidNumber: 2013

dn: cn=인사관리,ou=인사관리,ou=HR,ou=departments,dc=infra,dc=com
objectClass: posixGroup
cn: 인사관리
gidNumber: 2014
```

### 사용자 리스트 생성 (users.csv - 직접 관리)

```bash
uid,cn,sn,ou,uidNumber,gidNumber
user1,홍길동,홍,백엔드,3001,2001
user2,이몽룡,이,백엔드,3002,2001
user3,성춘향,성,디지털광고,3003,2004
user4,임꺽정,임,디지털광고,3004,2004
user5,변학도,변,콘텐츠기획,3005,2005
user6,김철수,김,국내영업,3006,2007
user7,박영희,박,국내영업,3007,2007
user8,최자영,최,해외영업,3008,2008
user9,정우성,정,서버관리,3009,2010
user10,강동원,강,네트워크관리,3010,2011
```

사용자 추가 csv 파일을 이용해 자동화

```
while IFS=, read -r uid cn sn ou uidNumber gidNumber; do
    ldapadd -x -D "cn=admin,dc=infra,dc=com" -W <<EOF
dn: uid=$uid,ou=$ou,ou=departments,dc=infra,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
cn: $cn
sn: $sn
uid: $uid
uidNumber: $uidNumber
gidNumber: $gidNumber
homeDirectory: /home/$uid
loginShell: /bin/bash
EOF
done < users.csv
```

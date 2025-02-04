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
1. 📌 오픈소스 및 표준 준수
- OpenLDAP은 GPL(GNU General Public License) 및 OpenLDAP Public License로 배포되며, 무료로 사용 및 수정 가능
- RFC 4510 및 관련 표준을 준수하여 다양한 LDAP 클라이언트 및 서버와 호환 가능

2. 🔐 강력한 인증 및 보안 기능
- SASL(Simple Authentication and Security Layer) 및 TLS/SSL(Transport Layer Security) 지원
- Kerberos 및 GSSAPI를 활용한 인증 연동 가능
- 액세스 제어 목록(ACL)을 사용한 세부적인 권한 관리

3. ⚡ 고성능 및 확장성
- 멀티 스레드 지원으로 고성능 처리 가능
- LDAP 복제(Replication) 기능을 통해 다중 서버 구성 가능
- 동적 구성(Dynamic Configuration, cn=config) 지원으로 실시간 변경 가능

4. 🏢 기업 환경 및 다양한 응용 가능
- 중앙 사용자 인증 시스템으로 활용 가능 (Linux PAM, Windows AD와 연동 가능)
- 메일 서버(Postfix, Dovecot) 및 웹 서비스(Apache, Nginx)와 연동 가능
- DNS, DHCP, RADIUS 등의 서비스와 연계 가능

## 구성요소
OpenLDAP은 여러 가지 핵심 구성 요소로 이루어져 있으며, 각 구성 요소는 디렉터리 서비스의 핵심 기능을 담당합니다.

1) slapd (Standalone LDAP Daemon)
- OpenLDAP 서버의 핵심 구성 요소로, 클라이언트의 요청을 처리
- LDAP 데이터베이스 관리, 인증, 접근 제어 기능 제공
- slapd.conf 또는 cn=config를 통해 설정 가능

2) ldapadd / ldapmodify / ldapdelete (CLI 도구)
- ldapadd: LDIF(LDAP Data Interchange Format) 파일을 이용하여 데이터 추가
- ldapmodify: 기존 LDAP 데이터를 수정
- ldapdelete: LDAP 객체 삭제

3) ldapsearch (LDAP 데이터 검색 도구)
- LDAP 디렉터리에서 정보를 검색하는 CLI 명령어
- 필터와 속성을 지정하여 원하는 정보 조회 가능

예제:
```bash
ldapsearch -x -LLL -b "dc=example,dc=com" "(objectClass=inetOrgPerson)"
```

4) slurpd / syncrepl (LDAP 복제 기능)
- slurpd: 과거에 사용되던 복제(replication) 방식 (현재는 사용되지 않음)
- syncrepl: 최신 OpenLDAP에서 사용되는 복제 기술로 마스터-슬레이브, 다중 마스터 복제 지원

5) ACL (Access Control List)
- 사용자 및 그룹별 읽기/쓰기/수정 권한 설정 가능
- olcAccess 또는 slapd.conf에서 설정 가능

예제:
```bash
access to dn.subtree="dc=example,dc=com"
    by self write
    by users read
    by anonymous auth
```

6) TLS/SSL 및 SASL 인증
- 보안 강화를 위해 TLS/SSL을 통한 암호화 통신 지원
- SASL (Simple Authentication and Security Layer)을 이용한 다양한 인증 방식 제공

예제 (TLS 활성화):
```bash
TLSCertificateFile /etc/ssl/certs/ldap.crt
TLSCertificateKeyFile /etc/ssl/private/ldap.key
```

📌 정리

✅ OpenLDAP은 중앙 인증 및 디렉터리 서비스 관리에 최적화된 오픈소스 솔루션
✅ LDAP 표준을 준수하며, 다양한 보안 및 인증 기능 제공
✅ 멀티 서버 환경을 지원하며, 기업 환경에서 널리 사용 가능
✅ CLI 도구 및 GUI(LAM, phpLDAPadmin)와 연동 가능하여 편리한 관리 가능

다음 단계에서는 OpenLDAP의 설치 방법, 기본 설정, 사용자 및 그룹 관리에 대해 다룰 예정입니다. 🚀

###

```bash
cat <<EOF >> /etc/hosts
10.1.1.100    ldap.
EOF
```

```bash
sudo apt update
sudo apt install slapd ldap-utils
```

```bash
sudo dpkg-reconfigure slapd
```

```bash
sed -i.bak -e '/^#URI[[:space:]]*ldap:\/\//a\
BASE   dc=localdomain,dc=com\
URI    ldap://ldap.localdomain.com' /etc/ldap/ldap.conf
```

```bash
systemctl enable --now slapd
systemctl status slapd
```

```bash
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:///
```

```bash
sudo slapcat
```

```bash
sudo apt install ldap-account-manager php-fpm
```

```bash
sudo a2enconf php*-fpm
```

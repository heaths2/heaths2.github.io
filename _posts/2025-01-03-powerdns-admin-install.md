---
title: PowerDNS-Admin 설치방법
author: G.G
date: 2025-01-03 20:50:00 +0900
categories: [Blog, Provisioning]
tags: [powerdns, powerdns-admin, bind9]
---

## 개요
PowerDNS-Admin은 PowerDNS 서버의 관리 인터페이스를 제공하는 웹 애플리케이션입니다. 이 도구는 PowerDNS의 설정을 직관적인 웹 UI를 통해 관리할 수 있게 해줍니다. PowerDNS-Admin은 사용자가 DNS 레코드를 쉽게 추가, 수정, 삭제할 수 있도록 도와주며, 여러 도메인과 DNS 설정을 관리할 수 있습니다.

## 특징
1. 직관적인 웹 UI
- PowerDNS-Admin은 PowerDNS 서버의 설정을 시각적이고 직관적인 웹 인터페이스로 관리할 수 있게 합니다.

2. 다중 사용자 지원:
- 여러 사용자 계정을 관리할 수 있어 다양한 권한 설정이 가능합니다.

3. 권한 기반 액세스 제어
- 특정 도메인이나 레코드에 대한 접근을 제한할 수 있는 권한 관리 기능이 포함되어 있습니다.

4. SQLite/PostgreSQL/MySQL 지원
- 데이터베이스로 SQLite, PostgreSQL, MySQL을 지원하여 다양한 환경에 맞춰 사용할 수 있습니다.

5. 로그인 기능
- 인증된 사용자만 PowerDNS 서버에 접근할 수 있도록 로그인 기능을 제공합니다.

6. API 지원
- RESTful API를 통해 자동화된 DNS 관리가 가능합니다.

## 구성요소
1. PowerDNS
- DNS 서비스를 제공하는 주요 서버. PowerDNS-Admin은 이를 관리하는 웹 애플리케이션입니다.

2. PowerDNS-Admin
- PowerDNS의 DNS 레코드와 설정을 관리하는 웹 애플리케이션.

3. 데이터베이스
- PowerDNS-Admin은 DNS 데이터와 사용자 정보를 저장하기 위해 SQLite, PostgreSQL, 또는 MySQL을 사용할 수 있습니다.

4. 웹 서버
- PowerDNS-Admin은 Flask 기반의 웹 애플리케이션으로, 웹 서버가 필요합니다. Apache 또는 Nginx와 같은 서버에서 호스팅할 수 있습니다.

5. 인증 및 보안
- 사용자 인증을 위해 필요한 보안 설정이 필요합니다 (예: SSL 설정).

## 환경구성
1. 로케일 설정
2. 패키지 설치

```bash
localectl set-locale LANG=en_US.UTF-8
```

```bash
sudo apt-get update
sudo apt-get install software-properties-common gnupg2 lsb-release curl
```

## 설치

## 참조
- [PowerDNS 공식 리포지토리](http://repo.powerdns.com/#ubuntu)
- [PowerDNS 설치 가이](https://docs.brconsulting.info/en/docs/network/powerdns/02-pdns-installation/)
- [PostgreSQL 공식 문서](https://www.postgresql.org/download/linux/ubuntu/)
- [NodeJS 공식 문서](https://deb.nodesource.com/node_20.x/pool/main/n/nodejs)

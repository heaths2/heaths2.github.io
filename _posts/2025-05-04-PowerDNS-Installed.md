---
title: PowerDNS & PowerDNS Admin Docker 설치
author: G.G
date: 2025-05-04 12:17 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, PowerDNS, PowerDNS-Admin]
---

## 📌 개요
PowerDNS는 유연하고 확장 가능한 오픈소스 DNS 서버이며, PowerDNS-Admin은 이를 위한 웹 기반 관리 인터페이스입니다.
이 문서는 Kubernetes(K3s) 환경에서 Helm Chart를 활용해 PowerDNS + PowerDNS-Admin 스택을 설치하고,
내부망 DNS 서버로 구성하는 과정을 담고 있습니다.

## 🧭 등장배경
- /etc/hosts 기반 수동 관리의 확장성 한계
- 내부망에서 독립된 DNS 인프라 필요성
- GUI 기반의 레코드 관리와 API 자동화를 고려한 선택
- Docker Compose → Helm Chart 기반 Kubernetes 전환 필요

## 🧩 주요 특징 및 구성 요소

| 구성 요소                      | 설명                                 |
| -------------------------- | ---------------------------------- |
| **PowerDNS Authoritative** | PostgreSQL Backend 기반 권한 있는 DNS 서버 |
| **PowerDNS-Admin**         | GUI 기반 웹 인터페이스 (API 지원 포함)         |
| **PostgreSQL**             | 레코드 메타데이터 저장소                      |
| **MetalLB**                | K3s 환경에서 LoadBalancer 타입의 외부 IP 제공 |
| **Helm**                   | 배포 자동화 및 재사용 가능한 Chart 관리 도구       |

## 🏗️ 아키텍처

```bash
[Browser]
   |
   | HTTP
   v
[MetalLB LoadBalancer: 172.16.0.242:8080]
   |
   v
[PowerDNS-Admin Pod] ---> [PowerDNS API: 8081]
                        |
                        v
                  [PowerDNS Pod: 53/tcp,udp]
                        |
                        v
                [PostgreSQL Pod (DB Backend)]
```

- **네트워크 포트 정리**

| 서비스            | 포트             | 설명            |
| -------------- | -------------- | ------------- |
| PowerDNS       | 53/tcp,udp     | DNS 서비스 기본 포트 |
| PowerDNS API   | 8081/tcp       | 관리용 REST API  |
| PowerDNS Admin | 8080 (→ 외부 80) | GUI 인터페이스     |
| PostgreSQL     | 5432/tcp       | 데이터베이스 연결     |

## 📁 파일 구조

PowerDNS-Admin
├── charts
├── [Chart.yaml](#chartyaml)
├── templates
│   ├── [deployment-postgresql.yaml](#deployment-postgresqlyaml)
│   ├── [deployment-powerdns-admin.yaml](#deployment-powerdns-adminyaml)
│   ├── [deployment-powerdns.yaml](#deployment-powerdnsyaml)
│   ├── [metallb-config.yaml](#metallb-config.yaml)
│   ├── [pvc-postgresql.yaml](#pvc-postgresqlyaml)
│   ├── [service-postgresql.yaml](#service-postgresqlyaml)
│   ├── [service-powerdns-admin.yaml](#service-powerdns-adminyaml)
│   ├── [service-powerdns.yaml](#service-powerdnsyaml)
└── [values.yaml](#valuesyaml)


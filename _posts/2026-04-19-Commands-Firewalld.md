---
title: Firewalld 사용법
author: G.G
date: 2026-04-19 14:06 +0900
categories: [Blog, Command]
tags: [Command, Firewall, nftables, iptables]
---

## 📘 개요 (Overview)

Firewalld는 Linux 시스템에서 네트워크 접근 제어를 수행하는 방화벽 관리 도구입니다.  
내부적으로는 nftables(또는 iptables)를 사용하지만, 운영자는 zone 기반 정책을 통해 보다 직관적으로 방화벽을 관리할 수 있습니다.

특히 운영 환경에서는 다음과 같은 이유로 Firewalld 사용이 권장됩니다.

- 동적 룰 적용 (서비스 중단 없이 정책 변경 가능)
- Zone 기반 정책 관리 (신뢰 구간별 접근 제어)
- 화이트리스트 기반 보안 구성 가능

## 📂 디렉토리 구조 (Tree 구조)

```bash
/etc/firewalld/
├── firewalld.conf
├── helpers
├── icmptypes
├── ipsets
├── lockdown-whitelist.xml
├── policies
├── services
└── zones
    └── public.xml
```

## ⚙️ 설치 방법

- 

```bash
# 명령어를 어떤 패키지가 제공하는지 찾는 명령어
dnf provides firewall-cmd

# 패키지 설치
dnf install -y firewall-cmd

# 서비스 시작 및 활성화
systemctl enable --now firewalld

# 상태 확인
firewall-cmd --state
```

- 전체 설정 확인

```bash
# 전체 설정 확인
firewall-cmd --list-all
```

- 특정 포트 허용

```bash
# 일시적 허용(재부팅 시 사라짐)
firewall-cmd --add-port=8080/tcp
# 영구 허용(재부팅 후에도 유지)
firewall-cmd --add-port=8080/tcp --permanent

# 적용
firewall-cmd --reload
```

- 특정 서비스 차단

```bash
# 기본 불필요 서비스 제거
firewall-cmd --remove-service=ssh --permanent
firewall-cmd --remove-service=cockpit --permanent

# 적용
firewall-cmd --reload
```

- 특정 서비스 허용

```bash
# 영구 허용(재부팅 후에도 유지)
firewall-cmd --add-service=http --permanent
firewall-cmd --add-service=https --permanent

# 적용
firewall-cmd --reload
```

- 특정 IP만 허용 (화이트리스트)

```bash
# Port 기준 적용
firewall-cmd --add-rich-rule='rule family="ipv4" source address="1.2.3.4/32" port port="22" protocol="tcp" accept' --permanent

# 서비스 기준 적용
firewall-cmd --add-rich-rule='rule family="ipv4" source address="1.2.3.4/32" service name="ssh" accept' --permanent

# 적용
firewall-cmd --reload
```

- 특정 IP만 차단

```bash
# Port 기준 적용
firewall-cmd --add-rich-rule='rule family="ipv4" destination address="5.6.7.8/32" port port="22" protocol="tcp" drop' --permanent

# 서비스 기준 적용
firewall-cmd --add-rich-rule='rule family="ipv4" destination address="5.6.7.8/32" service name="ssh" drop' --permanent

# 적용
firewall-cmd --reload
```

- 나가는 트래픽 제한

```bash
firewall-cmd --add-rich-rule='rule family="ipv4" destination address="8.8.8.8/32" drop' --permanent

# 적용
firewall-cmd --reload
```

- Firewalld 로그 활성화

```bash
# 현재 상태 확인
firewall-cmd --get-log-denied

# 로그 활성화
# | 옵션        | 의미            |
# | --------- | ------------- |
# | off       | 로그 안 남김 (기본값) |
# | all       | 모든 차단 로그      |
# | unicast   | 유니캐스트만        |
# | broadcast | 브로드캐스트만       |
# | multicast | 멀티캐스트만        |

firewall-cmd --set-log-denied=all

# 차단로그 확인
dmesg -wT
```

## 참고 자료

- [공식 문서](https://firewalld.org/)

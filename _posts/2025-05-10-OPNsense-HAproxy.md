---
title: OPNsense HAProxy 사용법
author: G.G
date: 2025-05-10 21:01 +0900
categories: [Blog, Provisioning ]
tags: [Provisioning , HAProxy, Load balansing]
---

## 📌 개요
이 문서는 리눅스 서버 보안을 위해 iptables를 사용하여 기본 방화벽 정책을 구성하는 방법을 설명합니다.

특히 SSH 접속과 ICMP(ping) 요청만 허용하고, 나머지 트래픽은 차단하는 화이트리스트 방식의 최소 보안 정책을 중심으로 구성되어 있습니다.

이 문서에서는 iptables 명령어의 주요 옵션 설명부터 시작해, 실제 적용 가능한 규칙 설정, 저장 및 복원, 정책 제거 방법까지 실무에 바로 적용할 수 있도록 예시와 함께 정리되어 있습니다.



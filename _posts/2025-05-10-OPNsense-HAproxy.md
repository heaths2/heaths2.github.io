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


![그림_1](/assets/img/2025-05-10/그림1.png)
_OPNsense HAProxy plugin 설치_

![그림_2](/assets/img/2025-05-10/그림2.png)
_OPNsense HAProxy 서비스 선택_

![그림_3](/assets/img/2025-05-10/그림3.png)
_OPNsense HAProxy 서비스 활성화_

![그림_4](/assets/img/2025-05-10/그림4.png)
_OPNsense HAProxy Health 모니터 선택_

![그림_5](/assets/img/2025-05-10/그림5.png)
_OPNsense HAProxy Health 서비스 포트 등록_

![그림_6](/assets/img/2025-05-10/그림6.png)
_OPNsense HAProxy 실제 서비스 할 서버 등록 선택_

![그림_7](/assets/img/2025-05-10/그림7.png)
_OPNsense HAProxy 실제 서비스 할 서버 등록_

![그림_8](/assets/img/2025-05-10/그림8.png)
_OPNsense HAProxy Backend 풀 등록 선택_

![그림_9](/assets/img/2025-05-10/그림9.png)
_OPNsense HAProxy Backend 풀 등록-1_

![그림_10](/assets/img/2025-05-10/그림10.png)
_OPNsense HAProxy Backend 풀 등록-2_

![그림_11](/assets/img/2025-05-10/그림11.png)
_OPNsense HAProxy Backend 풀 등록-3_

![그림_12](/assets/img/2025-05-10/그림12.png)
_OPNsense HAProxy Backend 풀 등록-4_

![그림_13](/assets/img/2025-05-10/그림13.png)
_OPNsense HAProxy Frontend 등록 선택_

![그림_14](/assets/img/2025-05-10/그림14.png)
_OPNsense HAProxy Frontend 등록-1_

![그림_15](/assets/img/2025-05-10/그림15.png)
_OPNsense HAProxy Frontend 등록-2_

![그림_16](/assets/img/2025-05-10/그림16.png)
_OPNsense HAProxy 설정 적용_

![그림_17](/assets/img/2025-05-10/그림17.png)
_OPNsense HAProxy 설정 확인_
---
title: OPNsense HAProxy 사용법
author: G.G
date: 2025-05-10 21:01 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, HAProxy, Load Balancing]
---

## 📘 개요 (Overview)
이 문서는 L4/L7 하드웨어 로드밸런서의 고비용 구조를 대체하고자, OPNsense의 HAProxy 플러그인을 활용하여 소규모 트래픽 환경에서 경제적이면서도 안정적인 로드밸런싱을 구현한 실제 운영 환경 기반 매뉴얼입니다.

외부에서 들어오는 HTTP 트래픽은 가상 IP (VIP: 10.1.1.101:80) 로 수신되며, 이를 OPNsense 내 HAProxy가 내부 WAS 서버(WAS1, WAS2)로 적절하게 분산 처리합니다.
복잡한 L7 정책 없이도 단순한 Round-Robin 방식으로 안정적인 서비스 운영이 가능하며, 장애 시에도 HA 구성된 방화벽 및 OPNsense 이중화로 고가용성을 확보합니다.

> 이 구성은 다음과 같은 환경에 적합합니다.
- 💸 고가의 상용 L4/L7 로드밸런서 대체
- 🧩 소규모 또는 개발/운영 통합 인프라
- 🖧 복잡하지 않은 HTTP 기반 트래픽 처리
- 🔁 단순한 라운드로빈 분산 및 헬스체크 필요 시

## 🧱 아키텍처 (Architecture)

```bash
[Client]
   |
   v
╔════════════╗
║  FW (EX)   ║  ← 외부 방화벽 (HA 구성)
╚════════════╝
   |
   v
╔═════════════════════════════════════╗
║ OPNsense (HA 구성, HAProxy 사용)   ║
║ Virtual IP: 10.1.1.101:80           ║
╚═════════════════════════════════════╝
   |
   v
╔═════════════════════════════════════╗
║ Internal FW (FW-IN, HA 구성)       ║
╚═════════════════════════════════════╝
   |
   v
╔═════════════════════════════════════╗
║           Internal Network          ║
║ ┌────────────┐   ┌────────────┐     ║
║ │ WAS1       │   │ WAS2       │     ║
║ │ 10.1.11.101│   │ 10.1.11.102│     ║
║ │   :8080    │   │   :8080    │     ║
║ └────────────┘   └────────────┘     ║
║                                     ║
║ ┌────────────┐   ┌────────────┐     ║
║ │ DB1        │   │ DB2        │     ║
║ │ (예: 5432) │   │ (예: 5432) │     ║
║ └────────────┘   └────────────┘     ║
╚═════════════════════════════════════╝
```

## ⚙️ 사용법

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
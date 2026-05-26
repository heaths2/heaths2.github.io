---
title: Cisco Switch FHRP 설정
author: G.G
date: 2025-01-08 08:00:00 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Cisco, FHRP, HSRP, VRRP, GLBP]
---

## 📘 개요
Cisco의 **FHRP(First Hop Redundancy Protocol)**는 네트워크의 첫 번째 홉(디폴트 게이트웨이)을 이중화하여 장애 상황에서도 안정적인 네트워크 연결을 보장하는 기술입니다. 이를 통해 네트워크 클라이언트는 특정 디폴트 게이트웨이의 장애에도 불구하고 정상적으로 라우터를 통해 외부 네트워크와 통신할 수 있습니다. Cisco FHRP의 대표적인 프로토콜에는 HSRP(Hot Standby Router Protocol), VRRP(Virtual Router Redundancy Protocol), 그리고 GLBP(Gateway Load Balancing Protocol)가 있습니다.

## 특징
1. 게이트웨이 리던던시
- 네트워크 클라이언트는 단일 IP를 디폴트 게이트웨이로 사용하며, 백업 라우터가 자동으로 역할을 인수합니다.
2. 라우터 장애 복구
- 활성 라우터가 장애를 일으킬 경우, 대기(Standby) 라우터가 자동으로 디폴트 게이트웨이 역할을 수행.
3. 가상 IP 주소
- 클라이언트는 실제 라우터 IP 주소가 아닌 가상 IP(Virtual IP)를 게이트웨이로 설정.
4. 프로토콜 지원
- HSRP: Cisco 독점 프로토콜로 라우터 간 기본 및 대기 역할 분리.
- VRRP: 산업 표준 프로토콜로 비슷한 기능을 제공.
- GLBP: Cisco 독점 프로토콜로 부하 분산 기능 추가.
5. 로드 밸런싱 (GLBP에서만 제공):
- 여러 라우터 간 트래픽 부하를 분산하여 네트워크 성능 최적화.

6. **FHRP 프로토콜 비교**

| **특징**                     | **HSRP**                              | **VRRP**                              | **GLBP**                              |
|------------------------------|---------------------------------------|---------------------------------------|---------------------------------------|
| **프로토콜 유형**             | Cisco 독점                           | 산업 표준 (IETF RFC 5798)             | Cisco 독점                           |
| **가상 IP 주소 사용**         | 사용                                  | 사용                                  | 사용                                  |
| **활성 라우터 (Active)**      | 단일 활성 라우터                     | 단일 활성 라우터                     | 다중 활성 라우터 가능                |
| **대기 라우터 (Standby)**     | 한 대의 대기 라우터 지정              | 한 대의 대기 라우터 지정              | 다수의 대기 라우터 지원              |
| **로드 밸런싱**               | 기본적으로 지원하지 않음 (그룹 2개로 구현 가능) | 기본적으로 지원하지 않음 (그룹 2개로 구현 가능) | 지원 (단일 게이트웨이로 자동 부하 분산) |
| **라우터 우선순위 설정**      | 지원 (기본값: 100)                   | 지원 (기본값: 100)                   | 지원 (기본값: 100)                   |
| **프리엠프 (Preemption)**     | 옵션 (기본값: 비활성화)               | 기본 활성화                           | 기본 활성화                           |
| **가상 MAC 주소**             | `0000.0C07.ACxx`                     | `0000.5E00.01xx`                     | `0007.B4xx.xxyy`                     |
| **트래픽 분산**               | 기본적으로 지원하지 않음 (그룹 2개로 분산 가능) | 기본적으로 지원하지 않음 (그룹 2개로 분산 가능) | 라운드 로빈 방식으로 자동 부하 분산   |
| **Hello 타이머 기본값**       | 3초                                  | 1초                                  | 3초                                  |
| **대기 타이머 기본값**         | 10초                                 | 3초                                  | 10초                                 |
| **구성 복잡성**               | 중간                                  | 낮음                                  | 높음                                  |
| **장점**                     | 단순 구성, 안정적인 장애 복구         | 산업 표준으로 벤더 독립적            | 부하 분산, 높은 가용성 제공           |
| **단점**                     | Cisco 장비에서만 사용 가능            | 로드 밸런싱 구현 시 추가 구성 필요     | Cisco 장비에서만 사용 가능, 복잡한 설정 |

## 구성요소
1. 가상 IP 주소 (VIP)
- 네트워크 클라이언트가 사용하는 디폴트 게이트웨이 IP. 실제 라우터가 장애 시에도 동일하게 유지.

2. 활성 라우터 (Active Router)
- 현재 디폴트 게이트웨이 역할을 수행하는 라우터.

3. 대기 라우터 (Standby Router)
- 활성 라우터 장애 시 디폴트 게이트웨이 역할을 대체하는 라우터.

4. 우선 순위 (Priority)
- 라우터 간 활성 및 대기 역할을 지정하는 기준. 기본값은 100이며, 높은 값을 가진 라우터가 활성 라우터로 설정됨.

5. 가상 MAC 주소
- 가상 IP에 연결된 MAC 주소로, 장애 발생 시 MAC 주소가 그대로 유지되어 네트워크가 끊기지 않음.

6. 프로토콜 메시지
- 라우터 간 상태 정보를 교환하기 위한 메시지. 예: HSRP의 Hello 메시지.


## VRRP 설정

### 단일 GW 설정
```bash
R1# configure terminal
R1(config)# interface vlan 100
R1(config-if)# vrrp 1 ip 10.1.1.1
R1(config-if)# vrrp 1 ip priority 120
R1(config-if)# vrrp 1 preempt
R1(config-if)# vrrp 1 track 10 decrement 30
```

```bash
R2# configure terminal
R2(config)# interface vlan 100
R2(config-if)# vrrp 1 ip 10.1.1.1
R2(config-if)# vrrp 1 preempt
```

### LB GW 설정

```bash
R1# configure terminal
R1(config)# interface vlan 100
R1(config-if)# vrrp 2 ip 10.1.1.2
R1(config-if)# vrrp 2 preempt
```

```bash
R2# configure terminal
R2(config)# interface vlan 100
R2(config-if)# vrrp 2 ip 10.1.1.2
R2(config-if)# vrrp 2 ip priority 120
R2(config-if)# vrrp 2 preempt
R2(config-if)# vrrp 2 track 10 decrement 30
```

### VRRP 상태 확인

```bash
R1# show vrrp all
R1# show vrrp brief
```

```bash
R2# show vrrp all
R2# show vrrp brief
```

## HSRP 설정

### 단일 GW 설정

```bash
R1# configure terminal
R1(config)# interface vlan 100
R1(config-if)# standby 1 ip 10.1.1.1
R1(config-if)# standby 1 ip priority 120
R1(config-if)# standby 1 preempt
R1(config-if)# standby 1 track Gi1/1 30
```

```bash
R2# configure terminal
R2(config)# interface vlan 100
R2(config-if)# standby 1 ip 10.1.1.1
R2(config-if)# standby 1 preempt
```

### LB GW 설정

```bash
R1# configure terminal
R1(config)# interface vlan 100
R1(config-if)# standby 2 ip 10.1.1.2
R1(config-if)# standby 2 preempt
```

```bash
R2# configure terminal
R2(config)# interface vlan 100
R2(config-if)# standby 2 ip 10.1.1.2
R2(config-if)# standby 2 ip priority 120
R2(config-if)# standby 2 preempt
R2(config-if)# standby 2 track Gi1/1 30
```

### HSRP 상태 확인

```bash
R1# show standby all
R1# show standby brief
```

```bash
R2# show standby all
R2# show standby brief
```

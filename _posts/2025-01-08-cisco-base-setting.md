---
title: Cisco Switch 기본 설정
author: G.G
date: 2025-01-08 09:00:00 +0900
categories: [Blog, Provisioning]
tags: [provisioning, cisco, ntp, snmp, banner, scp, acl, aaa model]
---

## 개요
Cisco 스위치는 네트워크의 중심 장비로, 보안 및 효율성을 높이기 위해 초기 설정을 신중히 수행해야 합니다. 이 매뉴얼은 스위치의 기본 설정 항목과 보안 및 관리에 필수적인 설정 방법을 제공합니다.

## 

### 호스트 이름 설정

```bash
Switch# configure terminal
Switch(config)# hostname SW
```

### ** Enable 및 사용자 비밀번호 설정**
1. Enable 비밀번호 (Scrypt 알고리즘)

```bash
Switch(config)# enable algorithm-type scrypt secret cisco
```

2. 사용자 계정 및 비밀번호

```bash
Switch(config)# username cisco privilege 15 algorithm-type scrypt secret cisco
```

### AAA 모델 비활성화

```bash
Switch(config)# no aaa new-model
```

### 도메인 이름 조회 비활성화

```bash
Switch(config)# no ip domain-lookup
```

### 시간대 및 NTP 서버 설정
1. 시간대 설정

```bash
Switch(config)# clock timezone KST 9 0
```

2. NTP 서버 추가

```bash
Switch(config)# ntp server 216.239.35.0
Switch(config)# ntp server 216.239.35.4
Switch(config)# ntp server 216.239.35.8
Switch(config)# ntp server 216.239.35.12
```

### 도메인 이름 및 SSH 설정
1. 도메인 이름 설정

```bash
Switch(config)# ip domain-name infra.com
```

2. SSH 키 생성 및 버전 설정

```bash
Switch(config)# crypto key generate rsa general-keys modulus 2048
```

```bash
Switch(config)# ip ssh version 2
Switch(config)# ip ssh time-out 120
```

3. SCP 활성화

```bash
Switch(config)# ip scp server enable
```

### SNMP 및 ACL 설정
1. SNMP ACL 생성

```bash
Switch(config)# ip access-list standard SNMP
Switch(config-std-nacl)# permit 10.1.1.100
```

2. SNMP 커뮤니티 설정

```bash
Switch(config)# snmp-server community nms_mailplug RO SNMP
```

### 로깅 서버 설정
1. 로깅 서버 추가

```bash
Switch(config)# logging host 10.1.1.100
```

2. 로깅 레벨 설정

```bash
Switch(config)# logging trap 7
# 또는
Switch(config)# logging trap debugging
```

### VTY 라인 설정
1. SSH 접근 제한 및 비활성화 타이머

```bash
Switch(config)# line vty 0 15
Switch(config-line)# transport input ssh
Switch(config-line)# login local
Switch(config-line)# exec-timeout 5 0
```

### SSH 공개 키 등록

```bash
Switch(config)# ip ssh pubkey-chain
Switch(conf-ssh-pubkey)# username cisco
Switch(conf-ssh-pubkey-user)# key-hash ssh-rsa 854581AA45BAD174CD00C8CEFFD1F959
```

### VLAN 설정
1. VLAN 생성 및 이름 지정

```bash
Switch(config)# vlan 100
Switch(config-vlan)# name Zone_10.1.1.0/24
```

2. VLAN 인터페이스 설정

```bash
Switch(config)# interface Vlan100
Switch(config-if)# ip address 10.1.1.10 255.255.255.0
```

### 기본 게이트웨이 설정

```bash
Switch(config)# ip default-gateway 10.1.255.1
```

### 포트 및 포트 채널 설정
1. 트렁크 포트 및 포트 채널 본딩

```bash
Switch(config)# interface range Gi1/0/47 - 48
Switch(config-if-range)# switchport mode trunk
Switch(config-if-range)# switchport trunk allowed vlan 100
Switch(config-if-range)# channel-group 1 mode active
```

2. 포트 설명 추가

```bash
Switch(config)# interface Gi1/0/47
Switch(config-if)# description bb-in-A02#Gi1/0/1
Switch(config)# interface Gi1/0/48
Switch(config-if)# description bb-in-A02#Gi2/0/1
```

3. 포트 채널 인터페이스 설명 추가

```bash
Switch(config)# interface Port-channel 1
Switch(config-if)# description bb-in-A02
```

```bash
Switch(config)# default-value exec-character-bits 8
Switch(config)# end
Switch# exit
```

### 배너 메시지 설정
1. **8비트 문자 세트(UTF-8)** 활성화
- 특수 문자나 한글 같은 비ASCII 문자를 정상적으로 표시하고 처리하기 위해 필수적인 설정.
- 7비트 ASCII 문자만 활성화된 기본 상태에서는 한글과 특수 문자 입력 시 오류 발생.
- **8비트 문자 세트(UTF-8)** 활성화 후 세션 종료 후 배너 등록 필수
  
```bash
Switch(config)# default-value exec-character-bits 8
Switch(config)# end
Switch# exit
```

2. 배너 메시지 설정

```bash
Switch(config)# banner motd ^
Enter TEXT message.  End with the character '^'.
+-----------------------------------------------------------------------+
|                                                                       |
|                        - 경          고 -                             |
|                                                                       |
|                  해당 시스템에 비인가자의 접근은 불법이며,            |
|                접근시 민/형사상의 처벌을 받을 수 있습니다.            |
|                                                                       |
|    해당 시스템에 모든 접속 시도 및 접속에 대해 상시 기록이 됩니다.    |
|                                                                       |
|                        - W A R N I N G -                              |
|                                                                       |
|      A notice that any unathorized use of the system is unlawful,     |
|    and may be subject to civil and/or criminal penalties.             |
|                                                                       |
|      Any of your attempt to connect to the system is logged.          |
|                                                                       |
+-----------------------------------------------------------------------+
^
```

### 설정 저장

```bash
Switch# write memory
```

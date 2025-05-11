---
title: Cisco Switch 기본 설정
author: G.G
date: 2025-01-08 09:00:00 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Cisco, NTP, SNMP, Banner, SCP, ACL, AAA Model]
---

## 📘 개요
Cisco 스위치는 네트워크의 중심 장비로, 보안 및 효율성을 높이기 위해 초기 설정을 신중히 수행해야 합니다. 이 매뉴얼은 스위치의 기본 설정 항목과 보안 및 관리에 필수적인 설정 방법을 제공합니다.

## 설정 요약 표

| **구분**                     | **명령어**                                                                                       | **설명**                                                                                            |
|-----------------------------|------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| **호스트 이름 설정**          | `hostname SW`                                                                                | 스위치의 호스트 이름을 `SW`로 설정.                                                               |
| **Enable 비밀번호 설정**      | `enable algorithm-type scrypt secret cisco`                                                 | Enable 모드 비밀번호를 Scrypt 알고리즘을 사용하여 `cisco`로 설정.                                 |
| **사용자 계정 생성**          | `username cisco privilege 15 algorithm-type scrypt secret cisco`                             | 사용자 `cisco` 계정을 Privileged 모드로 생성.                                                    |
| **AAA 모델 비활성화**         | `no aaa new-model`                                                                           | AAA 모델 비활성화.                                                                               |
| **도메인 이름 조회 비활성화** | `no ip domain-lookup`                                                                        | 잘못된 명령 입력 시 DNS 조회를 방지.                                                             |
| **시간대 설정**               | `clock timezone KST 9 0`                                                                     | 시간대를 KST로 설정.                                                                              |
| **NTP 서버 추가**             | `ntp server 216.239.35.x`                                                                    | NTP 서버 설정으로 시간 동기화.                                                                   |
| **도메인 이름 설정**          | `ip domain-name infra.com`                                                                   | 스위치의 도메인 이름을 설정.                                                                      |
| **SSH 활성화**               | `crypto key generate rsa general-keys modulus 2048` <br> `ip ssh version 2` <br> `ip ssh time-out 120` | SSH를 활성화하고 키를 생성하며, 인증 타임아웃을 설정.                                             |
| **SCP 활성화**               | `ip scp server enable`                                                                        | SCP 파일 전송 활성화.                                                                             |
| **SNMP 설정**                | `snmp-server community nms RO SNMP` <br> `ip access-list standard SNMP permit 10.1.1.100` | SNMP 커뮤니티와 ACL을 설정.                                                                       |
| **로깅 서버 설정**            | `logging host 10.1.1.100` <br> `logging trap debugging`                                       | 로깅 서버를 추가하고 디버깅 레벨 설정.                                                            |
| **VTY 라인 설정**             | `line vty 0 15` <br> `transport input ssh` <br> `login local` <br> `exec-timeout 5 0`        | SSH만 허용하고 5분 비활성 시 타임아웃 설정.                                                       |
| **SSH 공개 키 등록**          | `ip ssh pubkey-chain` <br> `username cisco` <br> `key-hash ssh-rsa [hash-value]`             | 공개 키 인증을 위한 사용자와 키 등록.                                                             |
| **VLAN 설정**                | `vlan 100` <br> `name Zone_10.1.1.0/24` <br> `interface Vlan100` <br> `ip address 10.1.1.10 255.255.255.0` | VLAN 생성 및 IP 설정.                                                                             |
| **기본 게이트웨이 설정**       | `ip default-gateway 10.1.255.1`                                                              | 디폴트 게이트웨이를 설정.                                                                         |
| **포트 채널 설정**            | `interface range Gi1/0/47 - 48` <br> `switchport mode trunk` <br> `channel-group 1 mode active` | 트렁크 포트를 본딩하여 Port-Channel로 설정.                                                       |
| **포트 설명 추가**            | `description bb-in-A02#Gi1/0/1`                                                              | 인터페이스에 설명을 추가하여 관리 용이.                                                           |
| **8비트 문자 세트 활성화**     | `default-value exec-character-bits 8`                                                       | 한글 및 특수 문자 처리를 위해 UTF-8 활성화.                                                      |
| **배너 메시지 설정**          | `banner motd`                                                                                | 한글 및 경고 메시지를 포함한 배너 생성.                                                          |
| **설정 저장**                | `write memory`                                                                               | 설정 내용을 저장.                                                                                 |

---

## 참고 사항
- 위 명령어는 **Cisco IOS** 기반 장비에서 테스트된 표준 설정입니다.
- 한글 배너 사용 시 반드시 `default-value exec-character-bits 8`을 먼저 설정한 뒤 재로그인해야 정상 동작합니다.
- 보안 강화 및 효율적인 관리가 필요할 경우 AAA 모델을 사용하는 것이 권장됩니다.

## 기본 설정

### 호스트 이름 설정

```bash
Switch# configure terminal
Switch(config)# hostname SW
```

### **Enable 및 사용자 비밀번호 설정**
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
Switch(config)# snmp-server community nms RO SNMP
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

### ACL 설정
1. ACL 정책 수립

```bash
Switch(config)# ip access-list standard ALLOW_LOGIN
Switch(config-std-nacl)# permit 10.1.1.0 0.0.0.255
Switch(config-std-nacl)# deny any log
```

2. Login 정책 적용

```bash
Switch(config)# line vty 0 15
Switch(config-line)# access-class ALLOW_LOGIN in
```

### Web UI 접근 비활성화 설정

```bash
Switch(config)# no ip http server
Switch(config)# no ip http secure-server
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

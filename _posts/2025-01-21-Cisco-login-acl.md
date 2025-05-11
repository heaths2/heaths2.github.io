---
title: Cisco Switch 로그인 제어 설정 개요
author: G.G
date: 2025-01-21 16:42:00 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Cisco, ACL, SSH]
---

## 📘 개요
이 문서는 Cisco 스위치에서 `ip ssh pubkey-chain` 명령어를 지원하지 않는 오래된 IOS 장비에서 SSH Key 인증을 사용할 수 없는 경우, IP 기반 접근 제어와 간소화된 인증 설정을 통해 최소한의 보안을 유지하며 장비를 관리하는 방법을 설명합니다. 이 설정은 긴급 상황이나 레거시 장비 유지 보수를 목적으로 사용되며, 내부망과 같은 신뢰할 수 있는 환경에서만 적용하는 것이 권장됩니다.

## 목적
1. SSH Key 인증 대체
- 오래된 Cisco IOS에서 SSH Key 인증을 지원하지 않을 경우, 관리 IP를 ACL로 제한하여 비슷한 수준의 접근 제어를 구현.
2. 접속 제어 강화
- 허용된 IP 주소만 스위치에 접근 가능하도록 설정.
- ACL로 인증되지 않은 IP의 접속을 차단하고 로그 기록.
3. 간소화된 인증 설정
- 빠른 접속을 위해 패스워드 인증을 생략하고 내부망에서 관리에 초점을 둔 설정.

## 설정 구성
1. AAA 활성화
- AAA (Authentication, Authorization, Accounting)를 활성화하여 인증 및 권한 부여를 제어합니다.
  - Authentication 설정:
    - 로그인 시 인증 과정을 생략하도록 설정.
  - Authorization 설정:
    - Exec 세션 권한을 로컬 사용자로 제어.
2. ACL로 IP 접근 제어
- 특정 IP 주소만 접근 가능하도록 ACL을 생성합니다.
3. VTY 라인에 ACL 적용
- ACL을 VTY 라인에 적용하여 SSH 접속 시 접근 제어를 수행합니다.
4. 설정 저장

```bash
Switch# configure terminal
Switch(config)# aaa new-model
Switch(config)# aaa authentication login default none
Switch(config)# aaa authorization exec default local
Switch(config)# ip access-list standard ALLOW_LOGIN
Switch(config-std-nacl)# permit 10.1.1.100
Switch(config-std-nacl)# permit 10.1.1.200
Switch(config-std-nacl)# deny any log
Switch(config-std-nacl)# exit
Switch(config)#line vty 0 15
Switch(config-line)#login authentication default 
Switch(config-line)#access-class ALLOW_LOGIN in
Switch(config-line)#end
Switch# write memory
```

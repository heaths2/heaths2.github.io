---
title: OPNsense 기본 세팅
author: G.G
date: 2025-04-20 21:08:00 +0900
categories: [Blog, Provisioning]
tags: [OPNsense]
---

## 📘 개요
Active Directory(AD)는 Microsoft에서 개발한 중앙 집중형 디렉터리 서비스로, 사용자 계정, 컴퓨터, 그룹, 네트워크 자원 등을 통합적으로 관리하기 위한 기업용 인프라입니다.
LDAP (Lightweight Directory Access Protocol) 은 이러한 디렉터리 정보를 질의하고 조작할 수 있도록 설계된 표준 프로토콜로, AD는 LDAP을 핵심 통신 수단으로 삼고 있으며, 이를 통해 타 시스템(Linux, Mac, 기타 LDAP 호환 시스템) 과의 상호운용성을 지원합니다. 

또한 AD는 Kerberos 프로토콜을 기반으로 한 SSO(단일 로그인) 기능을 제공함으로써, 보안성과 사용자 편의성을 동시에 충족시킵니다.

즉, LDAP은 AD의 핵심 통신 프로토콜이며, AD는 LDAP과 Kerberos를 융합하여 완성된 기업용 중앙 인증 및 자원 관리 시스템입니다.

이 문서에서는 PowerShell을 활용하여 이러한 AD 환경에 Windows 클라이언트를 자동으로 등록(도메인 가입)하고, 관리 표준에 따라 컴퓨터 이름을 설정하는 일련의 자동화 절차를 안내합니다.

## 🧬 Active Directory(AD)의 등장 배경
🔹왜 AD(Active Directory)가 등장했을까?
과거의 기업 네트워크 환경에서는 다음과 같은 문제가 있었습니다.

| 문제                           | 설명                                                      |
|------------------------------|-----------------------------------------------------------|
| 사용자 계정이 분산됨          | 부서별로 로컬 계정을 따로 관리 → 관리 복잡, 보안 취약    |
| 자원 접근 제어 어려움         | 각 서버에서 별도로 권한을 설정해야 함                    |
| 패스워드 정책/보안 정책 일관성 부족 | 중앙 통제 미비로 보안 정책이 부실                        |
| 로그/감사 관리 한계           | 사용자 추적 불가능 또는 어려움                            |


## 🧩 Active Directory의 주요 특징 및 구성 요소
| 구분                  | 설명                                                                 |
|---------------------|----------------------------------------------------------------------|
| 기반 프로토콜         | LDAP (디렉터리 질의), Kerberos (SSO 인증), DNS (도메인 탐색)         |
| 중앙 인증 서비스       | 사용자의 로그인, 컴퓨터 인증, 리소스 접근 권한을 중앙에서 제어       |
| 그룹 정책(GPO)        | 모든 클라이언트에 보안 정책, 네트워크 설정, 소프트웨어 배포 등을 자동 적용 |
| OU(조직 단위)          | 부서/기능별로 사용자 및 컴퓨터를 논리적으로 분류하여 관리             |
| 도메인 컨트롤러(DC)    | AD 서비스가 설치된 서버로 인증 및 디렉터리 질의 처리를 수행           |
| 포리스트(Forest)      | 하나 이상의 도메인을 포함하는 AD의 최상위 구조                        |
| 도메인(Domain)        | 동일한 보안 경계 내의 사용자, 그룹, 컴퓨터를 포함하는 논리 구조        |
| 트러스트(Trust)       | 서로 다른 도메인 간 인증 관계를 설정하여 상호 리소스 접근 허용         |
| 스키마(Schema)        | AD 객체(User, Computer 등)의 속성 정의 구조, 모든 객체의 설계도       |

## 사용방법
- ActiveDirectory 모듈설치

```bash
# 현재 컴퓨터가 속한 도메인 확인 (Workgroup 환경이면 "WORKGROUP"으로 나옴)
(Get-WmiObject Win32_ComputerSystem).Domain

# Active Directory 모듈이 설치되어 있는지 확인
Get-Module -ListAvailable ActiveDirectory

# RSAT(Remote Server Administration Tools) 설치 (Windows 10 이상에서 사용 가능)
# Active Directory 모듈 포함 도구 설치
Add-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
```

- AD 사용자 계정 비밀번호 변경 및 초기화

```bash
# 사용자 비밀번호 직접 변경 (reset이 아닌 교체)
Set-ADAccountPassword -Identity $Account `
    -OldPassword (ConvertTo-SecureString "pass1!" -AsPlainText -Force) `
    -NewPassword (ConvertTo-SecureString "pass1234!" -AsPlainText -Force)

# 계정 ID와 새 비밀번호 변수
$NewPassword = ConvertTo-SecureString "pass1!" -AsPlainText -Force

# 비밀번호 초기화
Set-ADAccountPassword -Identity $Account -Reset -NewPassword $NewPassword

# 비밀번호 변경 + 사용자 로그인 시 변경 요구 설정(강제 변경)
Set-ADUser -Identity $Account -ChangePasswordAtLogon $true
```

- AD 사용자 계정 잠금 해제

```bash
# 사용자 입력으로 계정 아이디 받기
$Account = "tester"
#$Account = Read-Host "조회할 AD 계정 ID를 입력하세요 (예: tester)"

# ✅ 로그인 실패 횟수 및 마지막 실패 시각 확인
Get-ADUser -Identity $Account -Properties BadLogonCount, LastBadPasswordAttempt |
    Select-Object Name, BadLogonCount, LastBadPasswordAttempt

# ✅ 계정이 잠겼는지 확인
Get-ADUser -Identity $Account -Properties LockedOut | Select-Object Name, LockedOut

# ✅ 계정 잠금 해제
Unlock-ADAccount -Identity $Account
```

## 참조
- [증권정보포털 세이브로](https://seibro.or.kr)

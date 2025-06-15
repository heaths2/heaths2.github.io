---
title: Migrating VirtualBox VM to Hyper-V 방법
author: G.G
date: 2025-06-15 15:11 +0900
categories: [Blog, Virtual Machine]
tags: [Virtual Machine, VirtualBox, Hyper-V, VDI, VHD]
---

## 📘 개요 (Overview)
- VirtualBox VM 디스크 형식 .vdi → Hyper-V 호환 .vhdx로 변환
- 변환된 디스크를 기반으로 Hyper-V VM 생성
- 이전 과정에서 발생할 수 있는 오류 해결 방법 안내
- 성능 최적화 및 네트워크 설정 가이드 제공

## 🏗️ 변환 절차 요약

```bash
.vdi (VirtualBox)
   │
   ▼ VBoxManage (CLI 또는 GUI)
.vhd
   │
   ▼ Convert-VHD (PowerShell)
.vhdx (Hyper-V 최적화 포맷)
   │
   ▼ Hyper-V에서 VM 생성
VM 부팅 성공
```

## ⚙️ 사용 방법

```bash
# .vdi → .vhd 변환 (VBoxManage)
cd C:\Program Files\Oracle\VirtualBox
VBoxManage clonemedium D:\VM\Rocky9.5\Rocky9.5.vdi D:\VM\Rocky9.5\Rocky9.5.vhd --format VHD

# 📌 관리자 권한 PowerShell 필수
Convert-VHD -Path D:\VM\worker02\worker02.vhd -DestinationPath D:\VM\worker02\worker02.vhdx
```

## 참고 자료
- [VirtualBox 공식 문서](https://www.virtualbox.org/manual/ch08.html#vboxmanage-clonemedium)
- [MS 공식 문서](https://learn.microsoft.com/en-us/powershell/module/hyper-v/convert-vhd)
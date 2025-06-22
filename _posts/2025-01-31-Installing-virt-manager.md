---
title: Virt-Manager 설치 매뉴얼
author: G.G
date: 2025-01-31 08:22:00 +0900
categories: [Blog, Virtual Machine]
tags: [Virtual Machine, Virt-Manager, Virsh, QEMU]
---

## 📘 개요
Virt-Manager(Virtual Machine Manager)는 KVM, QEMU 및 libvirt 기반 가상 머신을 관리하는 GUI 툴입니다. 이를 통해 가상 머신을 쉽게 생성, 관리 및 모니터링할 수 있다.

## 주요 기능
1. 가상 머신 생성 및 관리: KVM/QEMU 기반 VM을 GUI에서 생성하고 관리 가능
2. 리소스 모니터링: CPU, 메모리, 네트워크 사용량 확인
3. 원격 가상 머신 관리: SSH를 통해 원격 호스트의 VM 관리
4. 가상 머신 스냅샷: VM 상태 저장 및 복원 기능 제공

## 구성 요소
1. libvirt: 가상화 API로, KVM/QEMU, Xen, LXC 등을 관리하는 백엔드 서비스
2. QEMU/KVM: 가상 머신을 실행하는 하이퍼바이저
3. virt-manager: 가상 머신을 GUI 환경에서 관리할 수 있는 프론트엔드
4. virt-viewer: 가상 머신의 화면을 보기 위한 뷰어
5. virt-install: CLI 환경에서 가상 머신을 설치하는 명령줄 유틸리티

## 설치
1. 하드웨어/소프트웨어 가상화 지원 여부 확인
```bash
# Hardware
grep -E "vmx|svm" /proc/cpuinfo
# Software
lscpu | grep Virtualization
```

2. 패키지 설치
```bash
sudo apt update
sudo apt install virt-manager libvirt-daemon-system libvirt-clients libvirt-daemon-driver-xen ssh-askpass --no-install-recommends
```

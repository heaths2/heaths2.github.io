---
title: WSL 사용법
author: G.G
date: 2026-01-12 10:35 +0900
categories: [Blog, Command]
tags: [Command, Script, WSL, Windows]
---

## ⚙️ 명령어 사용법

### WSL 설치 및 📌 설치 확인

```bash
# Linux용 Windows 하위 시스템 사용
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# Virtual Machine 기능 사용
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# WSL 엔진 + Hyper-V 설치
wsl --install

# 커널 업데이트
wsl --update

# 설치된 Linux 목록
wsl --list --verbose
# 또는
# wsl -l -v

# 모든 WSL을 VM모드(WSL2)
# wsl --set-default-version <1|2>
wsl --set-default-version 2

# Microsoft 공식 이미지 조회
wsl  --list --online
# 또는
# wsl -l -o
```

```bash
# 이미지 배포
wsl --install --from-file D:\WSL\Rocky-9-WSL-Base.latest.x86_64.wsl --name Rocky9.7

# 특정 사용자로 서버 접속
wsl -d Rocky9.7 -u bob
```

### WSL에 USBIPD 설치

```bash
# 관리자 권한으로 실행
winget install --interactive --exact dorssel.usbipd-win
```

### USB 디바이스 연결

```bash
# 현재 연결된 USB 목록 보기
usbipd list

# 공유 허용
usbipd bind --busid 4-4

# WSL 연결
usbipd attach --wsl --busid 4-4

# 연결 해제
usbipd detach --busid 4-4
```

## 참고 자료
- [필수 패키지 설치](https://learn.microsoft.com/ko-kr/windows/wsl/install-manual)
- [공식 문서](https://learn.microsoft.com/ko-kr/windows/wsl/install)
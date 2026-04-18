---
title: WSL 사용법
author: G.G
date: 2026-01-12 10:35 +0900
categories: [Blog, Command]
tags: [Command, Script, WSL, Windows]
---

## ⚙️ 명령어 사용법

### WSL 설치 및 📌 설치 확인

![그림_1](/assets/img/2026-01-12/그림1.png)
_Windows 기능 켜기/끄기_

```bash
# Linux용 Windows 하위 시스템 사용
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
# Linux용 Windows 하위 시스템 기능 Enable 확인
dism /online /get-featureinfo /featurename:Microsoft-Windows-Subsystem-Linux
# Virtual Machine 기능 사용
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
# Virtual Machine 기능 Enable 확인
dism /online /get-featureinfo /featurename:VirtualMachinePlatform
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

# WSL 엔진 + Hyper-V 설치
# Ubuntu 기본설치
# wsl --install
# 이미지 배포
wsl --install --from-file D:\VM\WSL\Rocky-9-WSL-Base.latest.x86_64.wsl --name Rocky9 --version 2
```

```bash
# 특정 사용자로 서버 접속
wsl -d Rocky9 -u bob

# wsl -d Rocky9 -u bob 옵션 생략
# wsl /etc/resolv.conf 자동 생성 비활성화
cat << EOF >> /etc/wsl.conf

[user]
default=bob

[network]
generateResolvConf = false

[automount]
options = "metadata,umask=22,fmask=11"
EOF

cat << EOF > /etc/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF

# Python3 설치
sudo dnf install -y python3.12 python3.12-pip python3.12-devel
sudo python3.12 -m pip install --upgrade pip setuptools wheel

sudo alternatives --install /usr/bin/python python /usr/bin/python3.9 1
sudo alternatives --install /usr/bin/python python /usr/bin/python3.12 2

sudo alternatives --install /usr/bin/pip pip /usr/bin/pip3.9 1
sudo alternatives --install /usr/bin/pip pip /usr/bin/pip3.12 2

# cgroup V2 확인
podman info | grep cgroup
```

```bash
wsl --shutdown
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

### WSL MSI 설치

```bash
# WSL 명령어 오류시
curl -O https://github.com/microsoft/WSL/releases/download/2.6.3/wsl.2.6.3.0.x64.msi
```

## 참고 자료

- [WSL 설치 가이드](https://learn.microsoft.com/ko-kr/windows/wsl/install-manual)
- [WSL Git 공식주소](https://github.com/microsoft/WSL)

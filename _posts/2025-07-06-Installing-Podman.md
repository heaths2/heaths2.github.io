---
title: Podman & Podman Desktop 설치 가이드
author: G.G
date: 2025-07-06 13:07 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Podman, Podman Desktop]
---

## 📘 개요 (Overview)

Podman은 컨테이너를 관리하고 실행하기 위한 강력한 오픈 소스 도구이며, Podman Desktop은 이를 시각적으로 쉽게 관리할 수 있도록 돕는 GUI 애플리케이션입니다.

## 📌 목적 (Purpose)

- 🧭 등장배경: Docker와 같은 컨테이너 런타임이 널리 사용되면서, 데몬 없이 컨테이너를 실행하고 관리할 수 있는 대안의 필요성이 대두되었습니다. 특히 보안과 시스템 자원 활용 측면에서 데몬리스(daemonless) 방식의 컨테이너 솔루션이 주목받게 되었습니다.
- 🎯 기대효과: Podman 및 Podman Desktop을 통해 사용자는 데몬 없이 안전하게 컨테이너를 생성, 실행, 관리할 수 있으며, 직관적인 GUI를 통해 컨테이너 환경을 효율적으로 제어하여 개발 및 운영 생산성을 향상시킬 수 있습니다.

## 📝 구성 요소 (Components)

| 역할                         | 설명                                                                                           | 특징                                                                                                                                                     |
| ---------------------------- | ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Podman Engine**            | 컨테이너를 생성, 실행, 관리하는 핵심 CLI 도구입니다.                                           | 데몬 없이 컨테이너를 실행하며, 루트 권한 없이도 컨테이너를 관리할 수 있어 보안성이 높습니다. Docker CLI와 유사한 명령어를 사용하여 학습 곡선이 낮습니다. |
| **Podman Machine**           | Windows 및 macOS 환경에서 Podman 컨테이너를 실행하기 위한 경량 Linux 가상 머신입니다.          | 호스트 OS에 직접 컨테이너를 실행할 수 없는 환경에서 Podman Engine이 동작할 수 있도록 가상화된 환경을 제공합니다.                                         |
| **Podman Desktop**           | Podman 및 기타 컨테이너 런타임(예: Docker, Lima)을 시각적으로 관리하는 GUI 애플리케이션입니다. | 컨테이너, 이미지, 볼륨, 네트워크 등을 직관적인 UI로 확인하고 관리할 수 있습니다. 다양한 컨테이너 런타임을 통합하여 관리할 수 있는 편의성을 제공합니다.   |
| **Container Image Registry** | 컨테이너 이미지를 저장하고 공유하는 중앙 저장소입니다 (예: Docker Hub, Quay.io).               | Podman은 이 레지스트리에서 이미지를 가져오거나 푸시하여 컨테이너 배포 및 공유를 용이하게 합니다.                                                         |

## ⚙️ 설지방법

- 패키지 설치

```bash
# Podman & podman-compose 패키지 설치
sudo dnf install -y podman podman-compose

cat << EOF > /etc/containers/containers.conf
[engine]
compose_warning_logs = false
EOF

# podman 버전 확인
podman compose --version
```

- 기존 이미지 저장소 변경

```bash
# unqualified-search-registries 값 "docker.io"로 변경
sudo sed -i 's/^unqualified-search-registries = .*$/unqualified-search-registries = ["docker.io"]/' /etc/containers/registries.conf
```

- 기본 디렉토리 생성

```bash
# 디렉토리 생성
mkdir -pv /data/gitlab/{config,logs,data,backups}

# 데이터 디렉토리들에 개별 컨테이너 파일 컨텍스트 영구 적용 규칙 추가
sudo semanage fcontext -a -t container_file_t "/data(/.*)?"

# 영구 규칙 적용
sudo restorecon -Rv /data
```

{: .prompt-info }

> - 기본 저장소 위치 변경
>
> ```bash
> # Podman 기본 데이터 경로의 SELinux 컨텍스트 확인
> semanage fcontext -l|grep "/var/lib/containers"
>
> # 새로운 경로에 Podman 접근 권한을 위한 SELinux 컨텍스트 설정 및 적용
> semanage fcontext -a -e /var/lib/containers /data/podman/database
> restorecon -Rv /data/podman/database
>
> # 변경된 컨텍스트가 올바르게 적용되었는지 최종 확인
> semanage fcontext -l|grep "/var/lib/containers"
> semanage fcontext -l|grep "/data/podman/database"
> ```

```bash
# 컨테이너 실행 (백그라운드)
podman compose -f docker-compose.yml up -d

# Podman 4.6 이하 권장 Systemd Generate 방식
# 🛠️ 컨테이너 systemd 서비스 파일 생성
# podman generate systemd --name gitlab --files --new

# 📂 생성된 서비스 파일 시스템에 등록
# cp -v container-* /usr/lib/systemd/system/

podman generate systemd nginx > /etc/systemd/system/podman-nginx.service

# 🚀 서비스 활성화 및 즉시 시작
systemctl enable --now podman-nginx.service
```



## 참고 자료

- [공식 문서](https://podman.io/docs/installation)

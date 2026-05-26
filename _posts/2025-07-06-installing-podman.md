---
title: "Podman & Podman Desktop 설치 및 사용법 가이드"
author: G.G
date: 2025-07-06 13:07 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Podman, Podman Desktop, Container]
---

## 📘 개요 (Overview)

Podman은 데몬리스(daemonless) 구조를 채택한 컨테이너 런타임으로, Docker CLI와 호환되면서도 루트 권한 없이 안전하게 컨테이너를 운영할 수 있습니다. Podman Desktop은 이를 GUI로 통합 관리하는 도구입니다.

## 🧪 검증 환경 (Tested Environment)

| 항목 | 버전 / 정보 |
| --- | --- |
| **OS** | Rocky Linux 9.4 (x86_64) |
| **Kernel** | 5.14.0-427 |
| **Podman** | v5.x |
| **podman-compose** | v1.x |
| **검증일** | 2025-07-06 |
| **권한** | root |

## 📌 목적 (Purpose)

- 🧭 **등장배경**: Docker 데몬 의존성과 루트 권한 요구로 인한 보안/운영 리스크를 해소하기 위해 데몬리스 컨테이너 런타임 수요가 증가
- 🎯 **기대효과**: 루트리스 실행으로 보안 강화, systemd 통합으로 운영 일원화, Docker 명령어 호환으로 학습 비용 최소화

## 🧩 주요 구성 요소 (Components)

| 역할 | 설명 | 특징 |
| --- | --- | --- |
| **Podman Engine** | 컨테이너 생성/실행/관리 CLI | 데몬리스, 루트리스 지원, Docker CLI 호환 |
| **Podman Machine** | Win/macOS용 경량 Linux VM | 호스트 OS 가상화 계층 제공 |
| **Podman Desktop** | GUI 통합 관리 도구 | 다중 런타임(Podman/Docker/Lima) 통합 |
| **Image Registry** | 이미지 저장/공유 저장소 | Docker Hub, Quay.io 등 |

## ✅ 사전 요구사항 (Prerequisites)

| 항목 | 요구 사양 | 확인 명령어 |
| --- | --- | --- |
| **OS** | RHEL 9 계열 | `cat /etc/os-release` |
| **CPU** | 2 Core 이상 | `nproc` |
| **Memory** | 4GB 이상 | `free -h` |
| **SELinux** | enforcing 가능 | `getenforce` |
| **Dependencies** | jq, policycoreutils-python-utils | `which jq semanage` |

## 🛠️ 설치 방법 (Installation)

### 1단계: 패키지 설치

```bash
# Podman + compose 동시 설치
sudo dnf install -y podman podman-compose

# compose 경고 로그 비활성화 (선택)
cat << 'EOF' | sudo tee /etc/containers/containers.conf
[engine]
compose_warning_logs = false
EOF

# 버전 확인
podman --version
podman compose --version
```

### 2단계: 기본 이미지 저장소 변경

```bash
# unqualified-search-registries를 docker.io 단일로 변경
sudo sed -i 's/^unqualified-search-registries = .*$/unqualified-search-registries = ["docker.io"]/' \
  /etc/containers/registries.conf
```

### 3단계: 작업 디렉토리 구조 생성

```bash
# 서비스별 표준 디렉토리 생성
sudo mkdir -pv /opt/gitlab/{config,logs,data,backups}
sudo mkdir -pv /opt/{jenkins,nexus}

# SELinux 컨텍스트 등록 (컨테이너 마운트 허용)
sudo semanage fcontext -a -t container_file_t "/opt(/.*)?"
sudo restorecon -Rv /opt
```

{: .prompt-info }
> **기본 저장소 위치를 변경하려는 경우** SELinux 컨텍스트 이전(equivalence) 처리가 필요합니다.
> ```bash
> sudo semanage fcontext -a -e /var/lib/containers /data/podman/database
> sudo restorecon -Rv /data/podman/database
> ```

## 🔧 설정 (Configuration)

### 네트워크 대역 생성

```bash
# DevOps 전용 브릿지 네트워크
podman network create \
  --driver bridge \
  --subnet 10.90.0.0/24 \
  --gateway 10.90.0.1 \
  --dns 8.8.8.8 --dns 8.8.4.4 \
  --ip-range 10.90.0.64/26 \
  net_devops
```

### 이미지 사전 다운로드

```bash
podman pull docker.io/gitlab/gitlab-ce:latest
podman pull docker.io/jenkins/jenkins:latest-jdk21
podman pull docker.io/sonatype/nexus3:latest
```

## 🚀 검증 (Verification)

### 네트워크 / 볼륨 / 컨테이너 확인

```bash
# 1. 전체 네트워크 서브넷 일괄 확인
podman network inspect $(podman network ls -q) \
  | jq -r '.[] | "\(.name): \(.plugins[0].ipam.ranges[0][0].subnet // .subnets[0].subnet)"'

# 2. 특정 컨테이너 IP 확인
podman inspect <container> | jq '.[0].NetworkSettings.Networks'

# 3. 볼륨 마운트 경로 확인
podman volume inspect $(podman volume ls -q) \
  --format "table {{.Name}}\t{{.Mountpoint}}"

# 4. 컨테이너별 마운트 매핑
podman inspect <container> \
  --format '{{range .Mounts}}{{.Type}} | {{.Source}} -> {{.Destination}}{{println}}{{end}}'
```

## 🛠️ 운영 (Operations)

### Compose 기반 실행

```bash
# 백그라운드 실행
podman compose -f docker-compose.yml up -d

# 로그 추적
podman compose logs -f
```

### systemd 통합 (Podman 4.6 이하: generate 방식)

```bash
# 서비스 유닛 생성
podman generate systemd <container> > /etc/systemd/system/podman-<container>.service

# 활성화
sudo systemctl daemon-reload
sudo systemctl enable --now podman-<container>.service
```

### systemd 통합 (Podman 4.6 이상: Quadlet 방식 권장)

```bash
# 변환 스크립트 실행
bash podman-container-to-quadlet.sh --name gitlab --overwrite

# Dry-run 검증
/usr/libexec/podman/quadlet -dryrun
```

> Quadlet 변환 스크립트 전체 코드는 [Gist](#) 또는 별도 포스트 참조.

## 🩺 트러블슈팅 (Troubleshooting)

### Case 1: 볼륨 마운트 시 Permission denied

**원인**: SELinux 라벨 미부여

**해결**:
```bash
# 마운트 시 :Z 옵션 명시 (개별 컨테이너 전용 라벨)
podman run -v /opt/data:/data:Z ...

# 또는 호스트 디렉토리에 사전 라벨링
sudo chcon -Rt container_file_t /opt/data
```

### Case 2: registries.conf 변경 후 pull 실패

```bash
# 캐시 클리어
sudo rm -rf /var/lib/containers/cache

# 명시적 FQDN 사용 권장
podman pull docker.io/library/nginx:latest
```

## 참고 자료

- [Podman 공식 문서](https://podman.io/docs/installation)
- [Quadlet 공식 가이드](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
- [Red Hat: Rootless Containers](https://www.redhat.com/sysadmin/rootless-podman)

## 🏷️ 변경 이력 (Changelog)

| 날짜 | 버전 | 변경 내용 |
| --- | --- | --- |
| 2025-07-06 | 1.0 | 최초 작성 |
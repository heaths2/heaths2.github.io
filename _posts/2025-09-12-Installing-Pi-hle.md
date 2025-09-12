---
title: Pi-hole 설치 가이드
author: G.G
date: 2025-09-12 13:22 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Pi-hole]
---

## 📘 개요 (Overview)
Portainer는 컨테이너 환경을 웹 기반으로 시각화하여 관리하는 GUI 도구입니다. 이 가이드는 podman-compose를 사용하여 Portainer Community Edition(CE)을 설치하고, 기존에 실행 중인 서비스 컨테이너들과 연동하는 방법을 안내합니다.

## 📌 목적 (Purpose)
- 🧭 등장배경: 컨테이너 관리는 복잡하고 다양한 CLI(명령어 인터페이스) 명령어를 필요로 합니다. 컨테이너 상태, 로그, 볼륨, 네트워크 등의 정보를 한눈에 파악하기 어려워 초보자나 소규모 팀에게는 효율적인 관리가 어렵습니다. Portainer는 이러한 불편함을 해소하기 위해 등장했습니다.

* **🎯 기대효과**
* 직관적인 관리: 복잡한 CLI 명령 없이 마우스 클릭만으로 컨테이너를 쉽게 시작, 중지, 재시작할 수 있습니다.
* 가시성 향상: 시스템에서 실행 중인 모든 컨테이너와 리소스(CPU, 메모리) 사용 현황을 대시보드에서 한눈에 파악할 수 있습니다.
* 생산성 증대: 컨테이너 배포 및 모니터링 작업을 간편화하여 개발 및 운영 효율을 높일 수 있습니다.

## 📝 구성 요소 (Components)

| 역할 | 설명 | 특징 |
| :--- | :--- | :--- |
| **Portainer CE** | **Podman** 환경을 시각적으로 관리하는 웹 UI입니다. | 직관적인 **대시보드**를 제공하며, 복잡한 CLI 명령어를 대체합니다. 무료로 사용 가능한 버전입니다. |
| **Podman** | 컨테이너를 생성하고 실행하는 핵심 **런타임 엔진**입니다. | **경량화된 구조**와 **데몬리스(Daemon-less)** 방식을 특징으로 하며, **Docker와 높은 호환성**을 가집니다. |
| **`podman.socket`** | **Portainer**와 **Podman**이 서로 통신하는 데 사용하는 **API 통신 채널**입니다. | 시스템 서비스로 실행되며, **원격 API 연결**을 지원하여 **Portainer**에 컨테이너 관리 권한을 부여합니다. |
| **`podman-compose`** | `docker-compose.yml` 파일을 읽어 여러 컨테이너를 함께 관리하는 **설정 파일 관리 도구**입니다. | 컨테이너 설정의 **코드화**를 가능하게 하며, **배포 자동화**를 돕습니다. |
| **Bind Mount** | 호스트 서버의 특정 디렉터리를 컨테이너 내부에 연결하여 **데이터를 영구적으로 저장**하는 방식입니다. | 컨테이너를 삭제해도 데이터가 유지되며, 사용자가 직접 데이터 저장 경로를 지정할 수 있습니다. |

## ⚙️ 설치방법

- 🛠️ Podman 설치

```bash
# Podman & podman-compose 패키지 설치
sudo dnf install -y podman podman-compose

# podman-compose 설치 버전 확인
podman-compose --version
```

- 기존 이미지 저장소 변경

```bash
# unqualified-search-registries 값 "docker.io"로 변경
sudo sed -i 's/^unqualified-search-registries = .*$/unqualified-search-registries = ["docker.io"]/' /etc/containers/registries.conf
```

- Portainer 설치

```bash
# 포드맨 소켓이 활성화되었는지 확인
sudo systemctl enable --now podman.socket

# 디렉토리 생성
mkdir -pv /opt/portainer
mkdir -pv /data/portainer

# 데이터 디렉토리들에 개별 컨테이너 파일 컨텍스트 영구 적용 규칙 추가
sudo semanage fcontext -a -t container_file_t "/data(/.*)?"

# 영구 규칙 적용
sudo restorecon -Rv /data
```

```bash
# docker-compose 파일 생성
cat << EOF > /opt/portainer/docker-compose.yml
# /opt/portainer/docker-compose.yml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:lts
    container_name: portainer
    hostname: portainer
    restart: always
    privileged: true
    ports:
      - "8000:8000"
      - "9443:9443"
    volumes:
      - /run/podman/podman.sock:/var/run/docker.sock
      - /data/portainer:/data
      - /data/profile/.bashrc:/root/.bashrc   # bashrc 파일 매핑
EOF
```

```bash
# 설치 디렉토리로 이동
cd /opt/portainer

# Podman Compose를 이용한 컨테이너 실행 (백그라운드)
podman-compose up -d

# 🛠️ 컨테이너 systemd 서비스 파일 생성
podman generate systemd --name portainer --files --new

# 📂 생성된 서비스 파일 시스템에 등록
cp -v container-* /usr/lib/systemd/system/

# 🚀 서비스 활성화 및 즉시 시작
systemctl enable --now container-portainer
```

![그림_1](/assets/img/2025-09-06/그림1.png)
_Portainer 계정 생성_

![그림_2](/assets/img/2025-09-06/그림2.png)
_Portainer 시작 → Home_

![그림_3](/assets/img/2025-09-06/그림3.png)
_Portainer 컨테이너 정보_

![그림_4](/assets/img/2025-09-06/그림4.png)
_Portainer 컨테이너 목록_

![그림_5](/assets/img/2025-09-06/그림5.png)
_Portainer 도커 추가_

![그림_6](/assets/img/2025-09-06/그림6.png)
_Portainer 독립형 도커 선택_

```bash
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /:/host \
  portainer/agent:2.33.1
```

![그림_7](/assets/img/2025-09-06/그림7.png)
_Portainer Agent 추가_

![그림_8](/assets/img/2025-09-06/그림8.png)
_Portainer 추가된 도커 목록_

## 참고 자료
- [Portainer 공식 문서](https://docs.portainer.io/start/install-ce/server/podman/linux)

---
title: GitLab Community Edition 설치 가이드
author: G.G
date: 2026-04-01 18:05 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Docker, Podman, GitLab]
---

## 📘 개요 (Overview)

GitLab은 **소스 코드 관리(SCM, Source Code Management)**를 중심으로 협업 개발을 지원하는 통합 DevOps 플랫폼입니다. 개발자가 작성한 코드를 중앙 저장소에서 체계적으로 관리하고, 변경 이력을 추적하며, 여러 개발자가 동시에 안정적으로 협업할 수 있도록 도와줍니다.

또한 GitLab은 단순한 코드 저장소를 넘어, CI/CD(지속적 통합 및 배포) 기능을 통해 코드 변경 시 자동으로 빌드, 테스트, 배포까지 수행할 수 있는 환경을 제공합니다. 이를 통해 개발부터 운영까지의 전체 과정을 하나의 플랫폼에서 통합적으로 관리할 수 있습니다.

이 가이드는 GitLab Community Edition(CE)을 컨테이너 기반으로 설치하고, LDAP(Active Directory) 연동을 통해 사용자 인증을 통합하는 방법을 안내합니다.

## 📂 디렉토리 구조 (Tree 구조)

```bash
/opt/gitlab/
├── apps
│   └── java21-sample
│       └── docker-compose.yml
├── docker-compose.yml
│── .env
└── runners
    ├── build
    │   └── docker-compose.yml
    └── deploy
        └── docker-compose.yml

/data/gitlab/
├── backups
├── config
├── data
├── logs
└── runner-configs
```

## ⚙️ 설치방법

- 🛠️ Podman 설치

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
mkdir -pv /opt/gitlab/{apps,runners/{build,deploy}}
mkdir -pv /data/gitlab/{config,logs,data,backups,runner-configs}

# 데이터 디렉토리들에 개별 컨테이너 파일 컨텍스트 영구 적용 규칙 추가
sudo semanage fcontext -a -t container_file_t "/data(/.*)?"

# 영구 규칙 적용
sudo restorecon -Rv /data
```

```bash
# docker-compose 파일 생성

cat << EOF > /opt/gitlab/docker-compose.yml
# /opt/gitlab/docker-compose.yml
services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    container_name: gitlab
    restart: always
    hostname: gitlab
    environment:
      TZ: 'Asia/Seoul'
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://git.infra.local:8929'
        gitlab_rails['gitlab_shell_ssh_port'] = 2424

    ports:
      - '8929:8929'
      - '443:443'
      - '2424:22'
    volumes:
      # 요청하신 대로 /data/gitlab 하위 구조로 매핑
      - '/data/gitlab/config:/etc/gitlab'
      - '/data/gitlab/logs:/var/log/gitlab'
      - '/data/gitlab/data:/var/opt/gitlab'
      - '/data/gitlab/backups:/var/opt/gitlab/backups'
    shm_size: '256m'
    networks:
      net:
        ipv4_address: 10.90.0.100

networks:
  net:
    name: gitlab_net
    driver: bridge
    ipam:
      config:
        - subnet: 10.90.0.0/24
          ip_range: 10.90.0.100/30
          gateway: 10.90.0.1
EOF

cat cat << EOF > /opt/gitlab/gitlab-runner/build/docker-compose.yml
# /opt/gitlab/gitlab-runner/build/docker-compose.yml

services:
  gitlab-runner-build:
    image: 'gitlab/gitlab-runner:latest'
    container_name: gitlab-runner-build
    restart: always
    volumes:
      - /data/gitlab/runner-configs/build:/etc/gitlab-runner:Z
      - /run/podman/podman.sock:/var/run/docker.sock:Z
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
    environment:
      - TZ=Asia/Seoul
    networks:
      - net
    extra_hosts:
      - "git.infra.local:10.90.0.100"  # 고정된 서버 IP를 바라보게 설정

networks:
  net:
    name: gitlab_net
    external: true  # 위에서 생성된 네트워크를 공유
EOF

cat cat << EOF > /opt/gitlab/gitlab-runner/deploy/docker-compose.yml
# /opt/gitlab/gitlab-runner/deploy/docker-compose.yml

services:
  gitlab-runner-deploy:
    image: 'gitlab/gitlab-runner:latest'
    container_name: gitlab-runner-deploy
    restart: always
    volumes:
      - /data/gitlab/runner-configs/deploy:/etc/gitlab-runner:Z
      - /run/podman/podman.sock:/var/run/docker.sock:Z
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
    environment:
      - TZ=Asia/Seoul
    networks:
      - net
    extra_hosts:
      - "git.infra.local:10.90.0.100"  # 고정된 서버 IP를 바라보게 설정

networks:
  net:
    name: gitlab_net
    external: true  # 위에서 생성된 네트워크를 공유
EOF

cat << EOF > /opt/gitlab/.env
PASSWORD_AD=testing1234
EOF

podman exec -it gitlab-runner-build gitlab-runner register \
  --non-interactive \
  --url http://git.infra.local:8929 \
  --token glrt-hHyWVbnCHIt73kLogp6xZG86MQp0OjEKdToxCw.01.121ltcr96 \
  --executor docker \
  --docker-image eclipse-temurin:21-jdk-jammy

podman exec -it gitlab-runner-deploy gitlab-runner register \
  --non-interactive \
  --url http://git.infra.local:8929 \
  --token glrt-7rO4ZLXibG0l-M-OBTetn286MQp0OjEKdToxCw.01.1214ofaop \
  --executor shell

concurrent = 1
check_interval = 0
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "java21-build-runner"
  url = "http://git.infra.local:8929"
  id = 1
  token = "glrt-hHyWVbnCHIt73kLogp6xZG86MQp0OjEKdToxCw.01.121ltcr96"
  token_obtained_at = 2026-04-05T14:22:11Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "eclipse-temurin:21-jdk-jammy"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    volume_keep = false
    shm_size = 0
    network_mtu = 0
    extra_hosts = ["git.infra.local:10.90.0.100"]
concurrent = 1
check_interval = 0
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "java21-deploy-runner"
  url = "http://git.infra.local:8929"
  id = 2
  token = "glrt-7rO4ZLXibG0l-M-OBTetn286MQp0OjEKdToxCw.01.1214ofaop"
  token_obtained_at = 2026-04-05T14:22:24Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "shell"
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
```

```bash
# 컨테이너 실행 (백그라운드)
podman compose up -d

# LDAP 통신 확인
podman exec -it gitlab gitlab-rake gitlab:ldap:check

# 🛠️ 컨테이너 systemd 서비스 파일 생성
podman generate systemd --name gitlab --files --new

podman generate systemd nginx > /etc/systemd/system/podman-nginx.service

# 📂 생성된 서비스 파일 시스템에 등록
cp -v container-* /usr/lib/systemd/system/

# 🚀 서비스 활성화 및 즉시 시작
systemctl enable --now container-gitlab
```

```bash
podman exec -it gitlab-runner-01 gitlab-runner register \
  --non-interactive \
  --url http://git.infra.local:8929 \
  --token glrt-hHyWVbnCHIt73kLogp6xZG86MQp0OjEKdToxCw.01.121ltcr96 \
  --name java21-runner-01 \
  --executor docker \
  --docker-image eclipse-temurin:21-jdk-jammy
Runtime platform                                    arch=amd64 os=linux pid=15 revision=ac71f4d8 version=18.10.0
Running in system-mode.

Verifying runner... is valid                        correlation_id=01KNEFTYG1DQ8QXE0D4SWVVYZP runner=hHyWVbnCH runner_name=java21-runner-01
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!

Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml"
```


```bash
podman exec -it gitlab-runner-01 gitlab-runner list

podman exec -it gitlab-runner-01 gitlab-runner unregister --all-runners
```


![그림_1](/assets/img/2026-04-01/그림1.png)
_GitLab 계정 생성_

## 참고 자료

- [GitLab 공식 문서](https://docs.gitlab.com/install/docker/installation)
- [GitLab-LDAP 공식 문서](https://docs.gitlab.com/administration/auth/ldap/#configure-ldap)

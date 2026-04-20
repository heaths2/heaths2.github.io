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

## ⚙️ 설치방법

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
mkdir -pv /opt/gitlab/{config,logs,data,backups}
mkdir -pv /opt/{jenkins,nexus}

# 필요한 경로만 SELinux 컨텍스트 등록
semanage fcontext -a -t container_file_t "/opt/gitlab(/.*)?"
semanage fcontext -a -t container_file_t "/opt/jenkins(/.*)?"
semanage fcontext -a -t container_file_t "/opt/nexus(/.*)?"

# 적용
restorecon -Rv /opt/gitlab /opt/jenkins /opt/nexus
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

- **DevOps** 네트워크 대역 생성

```bash
podman network create \
  --driver bridge \
  --subnet 10.90.0.0/24 \
  --gateway 10.90.0.1 \
  --ip-range 10.90.0.64/26 \
  net_devops
```


```bash
# 컨테이너 네트워크 확인
podman network inspect $(podman network ls -q) | jq -r '.[] | "\(.name): \(.plugins[0].ipam.ranges[0][0].subnet // .subnets[0].subnet)"'

# 특정 컨테이너 IP 확인
podman inspect nginx | jq '.[0].NetworkSettings.Networks'
```

```bash
# 컨테이너 마운트 경로 확인
podman volume inspect $(podman volume ls -q) --format "table {{.Name}}\t{{.Mountpoint}}"

# 특정 컨테이너 마운트 경로 확인
podman inspect nginx --format '{{range .Mounts}}{{.Type}} | {{.Source}} -> {{.Destination}}{{println}}{{end}}'
podman inspect nginx --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'

# 저장소에서 컨테이너 이미지 가져오기
podman pull docker.io/gitlab/gitlab-ce:latest
podman pull docker.io/jenkins/jenkins:latest-jdk21
podman pull docker.io/sonatype/nexus3:latest

# 컨테이너 실행 (백그라운드)
podman compose -f docker-compose.yml up -d

# Dockerfile에서 VOLUME으로 선언된 경로를 출력
podman inspect jenkins/jenkins:latest-jdk21 --format '{{.Config.Volumes}}'

# Podman 4.6 이하 권장 Systemd Generate 방식
# 🛠️ 컨테이너 systemd 서비스 파일 생성
# podman generate systemd --name gitlab --files --new

# 📂 생성된 서비스 파일 시스템에 등록
# cp -v container-* /usr/lib/systemd/system/

podman generate systemd nginx > /etc/systemd/system/podman-nginx.service

# 🚀 서비스 활성화 및 즉시 시작
systemctl enable --now podman-nginx.service
```

````bash
# Podman 4.6 이하 권장 Systemd Quadlet 방식
bash podman-container-to-quadlet.sh --name gitlab --overwrite
# 디버깅 Dry-run 재확인
/usr/libexec/podman/quadlet -dryrun

```bash
#!/usr/bin/env bash
# -*- coding: utf-8 -*-
#
# Author: OpenAI & User Collaboration
# Version: 1.2.0
# Description: 기존 실행 중인 Podman 컨테이너를 Quadlet(.container) 파일로 변환합니다.

set -Eeuo pipefail

SCRIPT_NAME="$(basename "$0")"
CONTAINER_NAME=""
OUTPUT_DIR=""
ROOTLESS_MODE="false"
RELOAD_SYSTEMD="false"
OVERWRITE="false"
DEBUG_MODE="false"

# -----------------------------
# Color & Logging
# -----------------------------
if [[ -t 1 ]]; then
    C_RESET='\033[0m'; C_RED='\033[31m'; C_GREEN='\033[32m'; C_YELLOW='\033[33m'; C_BLUE='\033[34m'; C_MAGENTA='\033[35m'; C_CYAN='\033[36m'; C_BOLD='\033[1m'
else
    C_RESET=''; C_RED=''; C_GREEN=''; C_YELLOW=''; C_BLUE=''; C_MAGENTA=''; C_CYAN=''; C_BOLD=''
fi

timestamp() { TZ=Asia/Seoul date '+%Y-%m-%d %H:%M:%S %Z'; }
log_info() { printf '%b[%s] [INFO] %s%b\n' "${C_GREEN}" "$(timestamp)" "$*" "${C_RESET}" >&2; }
log_error() { printf '%b[%s] [ERROR] %s%b\n' "${C_RED}" "$(timestamp)" "$*" "${C_RESET}" >&2; }
die() { log_error "$*"; exit 1; }

usage() {
    cat <<EOF
사용법:
  ${SCRIPT_NAME} <컨테이너이름> [옵션]
  ${SCRIPT_NAME} --name <컨테이너이름> --reload --overwrite

옵션:
  --name <name>       변환할 컨테이너 이름
  --output-dir <dir>  생성될 .container 파일 저장 경로
  --user              Rootless 모드 사용 (~/.config/containers/systemd)
  --reload            생성 후 systemctl daemon-reload 실행
  --overwrite         기존 파일이 있을 경우 덮어쓰기
  -h, --help          도움말 출력
EOF
}

require_cmd() {
    command -v "$1" >/dev/null 2>&1 || die "필수 명령어가 없습니다: $1 (설치 후 다시 시도하세요)"
}

parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            --name) CONTAINER_NAME="$2"; shift 2 ;;
            --output-dir) OUTPUT_DIR="$2"; shift 2 ;;
            --user) ROOTLESS_MODE="true"; shift ;;
            --reload) RELOAD_SYSTEMD="true"; shift ;;
            --overwrite) OVERWRITE="true"; shift ;;
            -h|--help) usage; exit 0 ;;
            *) if [[ -z "${CONTAINER_NAME}" ]]; then CONTAINER_NAME="$1"; shift; else die "알 수 없는 인자: $1"; fi ;;
        esac
    done
    [[ -n "${CONTAINER_NAME}" ]] || die "컨테이너 이름을 입력해야 합니다."
}

# -----------------------------
# Extraction Logic
# -----------------------------

get_restart_policy() {
    local p; p="$(podman inspect --format '{{.HostConfig.RestartPolicy.Name}}' "${CONTAINER_NAME}" 2>/dev/null || true)"
    case "${p}" in always|unless-stopped) echo "always" ;; on-failure) echo "on-failure" ;; *) echo "no" ;; esac
}

write_quadlet() {
    local out_dir="$1" file_path="$2"
    local image restart_policy hn shm

    # 1. 이미지 및 기본 정보 추출
    image="$(podman inspect --format '{{.Config.Image}}' "${CONTAINER_NAME}")"
    # 이미지 이름 정규화 (Warning 방지)
    [[ "${image}" != *"/"* ]] && image="docker.io/library/${image}"

    restart_policy="$(get_restart_policy)"
    hn="$(podman inspect --format '{{.Config.Hostname}}' "${CONTAINER_NAME}" 2>/dev/null || true)"
    shm="$(podman inspect --format '{{.HostConfig.ShmSize}}' "${CONTAINER_NAME}" 2>/dev/null || echo "67108864")"

    mkdir -p "${out_dir}"
    [[ -e "${file_path}" && "${OVERWRITE}" != "true" ]] && die "파일이 이미 존재합니다: ${file_path} (--overwrite 사용)"

    log_info "변환 시작: ${CONTAINER_NAME} -> ${file_path}"

    {
        printf '# Auto-generated Quadlet file\n'
        printf '# Generated at: %s\n\n' "$(timestamp)"

        printf '[Unit]\n'
        printf 'Description=Quadlet Service for %s\n' "${CONTAINER_NAME}"
        printf 'After=network-online.target\n\n'

        printf '[Container]\n'
        printf 'Image=%s\n' "${image}"
        printf 'ContainerName=%s\n' "${CONTAINER_NAME}"

        # [수정] N 대문자 준수
        [[ -n "${hn}" && "${hn}" != "$(uname -n)" ]] && printf 'HostName=%s\n' "${hn}"

        # [수정] 환경변수 추출 (따옴표 처리 강화)
        podman inspect "${CONTAINER_NAME}" | jq -r '.[0].Config.Env // [] | .[]' | while read -r env_line; do
            # 1. 환경변수 라인이 비어있거나(=만 있는 경우 포함) 체크
            [[ -z "${env_line}" || "${env_line}" == "=" ]] && continue

            # 2. KEY=VALUE 형태에서 VALUE가 비어있는지 체크 (정교한 필터링)
            if [[ "${env_line}" == *"="* ]]; then
                val="${env_line#*=}" # '=' 이후의 값 추출
                [[ -z "${val}" ]] && continue
            fi

            printf 'Environment="%s"\n' "${env_line}"
        done

        # 포트 추출
        podman inspect "${CONTAINER_NAME}" | jq -r '
            .[0].HostConfig.PortBindings // {} | to_entries[] | .key as $cp | .value[]
            | if .HostIp != "" and .HostIp != "0.0.0.0" then "\(.HostIp):\(.HostPort):\($cp)" else "\(.HostPort):\($cp)" end
        ' | while read -r p; do printf 'PublishPort=%s\n' "$p"; done

        # [수정] 볼륨 추출 및 :Z 자동 권장
        podman inspect "${CONTAINER_NAME}" | jq -r '
            .[0].Mounts // [] | .[] | select(.Type == "bind") | "\(.Source):\(.Destination)"
        ' | while read -r v; do
            [[ "$v" != *":Z" && "$v" != *":z" ]] && v="${v}:Z"
            printf 'Volume=%s\n' "$v"
        done

        # ShmSize 변환 (bytes -> MB)
        if [[ "$shm" != "67108864" ]]; then
            printf 'ShmSize=%sm\n' "$((shm / 1024 / 1024))"
        fi

        printf '\n[Service]\n'
        [[ "${restart_policy}" != "no" ]] && printf 'Restart=%s\n' "${restart_policy}"

        printf '\n[Install]\n'
        printf 'WantedBy=multi-user.target\n'
    } > "${file_path}"
}

# -----------------------------
# Execution
# -----------------------------

main() {
    parse_args "$@"
    require_cmd podman
    require_cmd jq

    if ! podman container exists "${CONTAINER_NAME}"; then die "컨테이너가 존재하지 않습니다: ${CONTAINER_NAME}"; fi

    if [[ -z "${OUTPUT_DIR}" ]]; then
        if [[ "${ROOTLESS_MODE}" == "true" ]]; then
            OUTPUT_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/containers/systemd"
        else
            OUTPUT_DIR="/etc/containers/systemd"
        fi
    fi

    local file_path="${OUTPUT_DIR}/${CONTAINER_NAME}.container"
    write_quadlet "${OUTPUT_DIR}" "${file_path}"

    if [[ "${RELOAD_SYSTEMD}" == "true" ]]; then
        local cmd="systemctl daemon-reload"
        [[ "${ROOTLESS_MODE}" == "true" ]] && cmd="systemctl --user daemon-reload"
        $cmd && log_info "명령 실행 완료: $cmd"
    fi

    log_info "모든 작업이 완료되었습니다!"
    echo -e "\n${C_BOLD}${C_BLUE}생성된 파일 내용 확인:${C_RESET}"
    cat "${file_path}"
}

main "$@"
````

## 참고 자료

- [공식 문서](https://podman.io/docs/installation)

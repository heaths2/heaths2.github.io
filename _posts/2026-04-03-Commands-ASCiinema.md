---
title: ASCiinema 사용법
author: G.G
date: 2026-04-03 20:06 +0900
categories: [Blog, Command]
tags: [Command, Certbot]
---

## 📘 개요 (Overview)

Asciinema는 콘솔 및 SSH 환경에서 수행되는 사용자 작업을 기록하는 세션 기록 도구로, 운영자의 모든 명령어 입력과 실행 결과를 그대로 저장할 수 있습니다. 이를 통해 시스템에 대한 접근과 작업 이력을 명확하게 추적할 수 있으며, 보안 및 운영 관점에서 중요한 감사 로그(Audit Logging)와 감사 추적(Audit Trail)을 구현하는 데 활용됩니다.

특히 베스천(Bastion) 서버와 같은 중앙 접속 구간에서 Asciinema를 적용하면, 사용자 활동 기록(User Activity Recording)을 체계적으로 수집할 수 있어 내부 통제, 사고 대응, 보안 감사 대응(ISMS 등)에 효과적으로 활용할 수 있습니다. 또한 기록된 세션은 재생 가능한 형태로 제공되어 작업 검증, 장애 분석, 운영 표준화 및 교육 자료로도 활용할 수 있는 장점을 가집니다.

## 📂 디렉토리 구조 (Tree 구조)

```bash
# Console 및 SSH 로그인 자동 기록
/etc/profile.d/
├── bash_completion.sh
└── which2.sh

# Cron 작업 90일 이상 경과 파일 제거
/etc/cron.d
└── asciinema-cleanup
```

## ⚙️ 설지방법

- ASCiinema 설치

```bash
python3 -m pip install asciinema
```

- Console 및 SSH 감사 로그 / 감사 추적 설정

```bash
sudo cat > /etc/profile.d/asciinema-record.sh <<'EOF'
#!/usr/bin/env bash
# -*- coding: utf-8 -*-

# 1. 재귀 실행 및 비대화형 세션 방지
[[ -n "${ASCIINEMA_REC:-}" ]] && return 0 2>/dev/null
case "$-" in *i*) ;; *) return 0 2>/dev/null ;; esac

# 2. 필수 바이너리 및 TTY 체크
ASCIINEMA_BIN="$(command -v asciinema 2>/dev/null || true)"
TTY_PATH="$(tty 2>/dev/null || true)"
[[ -z "${ASCIINEMA_BIN}" || ! -x "${ASCIINEMA_BIN}" ]] && return 0 2>/dev/null
[[ -z "${TTY_PATH}" || "${TTY_PATH}" == "not a tty" ]] && return 0 2>/dev/null
[[ -n "${SSH_ORIGINAL_COMMAND:-}" ]] && return 0 2>/dev/null

# --- 경로 및 변수 설정 ---
RECORD_ROOT="/var/log/asciinema"
USER_NAME="${USER:-unknown}"
USER_DIR="${RECORD_ROOT}/${USER_NAME}"
TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
HOSTNAME_SHORT="${HOSTNAME:-$(hostname 2>/dev/null || echo unknown)}"

REMOTE_IP="local"
[[ -n "${SSH_CONNECTION:-}" ]] && REMOTE_IP="$(echo "${SSH_CONNECTION}" | awk '{print $1}')"

TTY_NAME="${TTY_PATH#/dev/}"
TTY_NAME="${TTY_NAME//\//_}"
RECORD_FILE="${USER_DIR}/${TIMESTAMP}_${HOSTNAME_SHORT}_${REMOTE_IP}_${TTY_NAME}.cast"

# ---------------------------------------------------------
# 3. 디렉터리 자동 생성 및 시니어 권한 설계 (보안 강화)
# ---------------------------------------------------------
# 로그 루트 생성 및 권한 설정
if [[ ! -d "${RECORD_ROOT}" ]]; then
    mkdir -p "${RECORD_ROOT}" 2>/dev/null
    chown root:root "${RECORD_ROOT}"
    chmod 1777 "${RECORD_ROOT}"
fi

# 사용자별 디렉터리 생성 (사용자가 삭제하지 못하도록 root 소유로 생성)
if [[ ! -d "${USER_DIR}" ]]; then
    mkdir -p "${USER_DIR}" 2>/dev/null
    # 소유권은 root, 그룹은 사용자에게 부여하여 '쓰기'는 가능하나 '폴더 삭제'는 불가하게 설정
    chown root:"${USER_NAME}" "${USER_DIR}" 2>/dev/null
    chmod 770 "${USER_DIR}" 2>/dev/null
fi

# ---------------------------------------------------------
# 4. 보안 경고 배너 출력 (경각심 고취)
# ---------------------------------------------------------
clear
echo -e "\033[1;31m"
echo "==============================================================="
echo "                [ SECURITY WARNING & NOTICE ]"
echo "==============================================================="
echo -e "\033[0m"
echo -e "본 서버는 내부 보안 정책에 따라 \033[1;33m모든 터미널 작업이 실시간 녹화\033[0m됩니다."
echo -e "접속 IP   : ${REMOTE_IP}"
echo -e "로그 경로 : ${RECORD_FILE}"
echo ""
echo -e "모든 명령어 입력 및 출력 결과는 감사 로그로 보관되며,"
echo -e "비인가된 활동은 관련 법규에 따라 처벌받을 수 있습니다."
echo -e "\033[1;31m"
echo "==============================================================="
echo -e "\033[0m"
sleep 1

# 5. 녹화 실행
export ASCIINEMA_REC=1
umask 022  # 생성되는 로그 파일 권한 제어

LOGIN_SHELL="${SHELL:-/bin/bash}"
[[ -x "${LOGIN_SHELL}" ]] || LOGIN_SHELL="/bin/bash"

exec "${ASCIINEMA_BIN}" rec \
    --quiet \
    --idle-time-limit 2 \
    --command "${LOGIN_SHELL} -l" \
    "${RECORD_FILE}"
EOF

# 1) 로그 저장소 초기화
sudo mkdir -p /var/log/asciinema
sudo chown root:root /var/log/asciinema
sudo chmod 1777 /var/log/asciinema

# 2. 크론 작업 내용을 파일로 생성 (root 권한 필요)
sudo tee /etc/cron.d/asciinema-cleanup <<'EOF'
# 매일 새벽 2시에 90일이 지난 asciinema 로그(.cast) 삭제
0 * * * * root /usr/bin/find /var/log/asciinema -type f -name "*.cast" -mtime +90 -delete
EOF

# 3. 파일 권한 설정 (보안상 644 권한 권장)
sudo chmod 644 /etc/cron.d/asciinema-cleanup
```

## 참고 자료

- [Asciinema 공식 문서](https://docs.asciinema.org/getting-started/)

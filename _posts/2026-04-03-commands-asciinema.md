---
title: ASCiinema 사용법
author: G.G
date: 2026-04-03 20:06 +0900
categories: [Blog, Command]
tags: [Command, ASCiinema]
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

## ⚙️ 명령어 사용법

- ASCiinema 설치

```bash
python3 -m pip install asciinema
```

- Console 및 SSH 감사 로그 / 감사 추적 설정

```bash
#!/usr/bin/env bash
# =============================================================================
# Script  : asciinema-record.sh
# Version : 1.1.0
# Date    : 2026-05-27
# =============================================================================
# Description:
#   SSH/로컬 로그인 시 터미널 세션을 asciinema 로 자동 녹화하는 감사 로그 스크립트.
#   /etc/profile.d/ 에 배치하여 모든 대화형 로그인 셸에서 자동 실행된다.
#
# Bugs Fixed:
#   [BUG-1] --command "bash -l": 로그인 셸 재귀 기동으로 asciinema 이중 실행
#            수정: -l 플래그 제거, 부모 PID 이중 가드 추가
#   [BUG-2] chown root:USER 비루트 실패(2>/dev/null 로 은폐): USER_DIR 권한 설정 무력화
#            수정: EUID 분기 처리, 관리자 사전 생성 안내 주석 추가
#
# Changelog:
#   1.1.0 (2026-05-27) - BUG-1/2 패치, 코드 정리
#   1.0.0              - 최초 작성
# =============================================================================

# ---------------------------------------------------------------------------
# 가드 A: asciinema v2+ 가 자동 설정하는 환경변수 확인
# ---------------------------------------------------------------------------
[[ -n "${ASCIINEMA_REC:-}" ]] && return 0 2>/dev/null

# 비대화형 셸 즉시 리턴
case "$-" in *i*) ;; *) return 0 2>/dev/null ;; esac

# SSH 원격 명령 직접 실행은 녹화 불필요
[[ -n "${SSH_ORIGINAL_COMMAND:-}" ]] && return 0 2>/dev/null

# ---------------------------------------------------------------------------
# 가드 B: 부모 프로세스가 asciinema 인지 확인
# [BUG-1] 환경변수 미전달 케이스(일부 OS/버전) 에서도 이중 실행 차단
# ---------------------------------------------------------------------------
_asciinema_check_parent() {
    local ppid_comm
    ppid_comm="$(cat /proc/"${PPID}"/comm 2>/dev/null)" || return 1
    [[ "${ppid_comm}" == "asciinema" ]]
}
_asciinema_check_parent && return 0 2>/dev/null
unset -f _asciinema_check_parent

# ---------------------------------------------------------------------------
# 필수 바이너리 및 TTY 확인
# ---------------------------------------------------------------------------
_AR_BIN="$(command -v asciinema 2>/dev/null)"
[[ -z "${_AR_BIN}" || ! -x "${_AR_BIN}" ]] && return 0 2>/dev/null

_AR_TTY="$(tty 2>/dev/null)"
[[ -z "${_AR_TTY}" || "${_AR_TTY}" == "not a tty" ]] && return 0 2>/dev/null

# ---------------------------------------------------------------------------
# 경로 및 변수 설정
# ---------------------------------------------------------------------------
_AR_ROOT="/var/log/asciinema"
_AR_USER="${USER:-$(id -un 2>/dev/null)}"
_AR_UDIR="${_AR_ROOT}/${_AR_USER}"
_AR_TS="$(date +%Y%m%d-%H%M%S)"

# FQDN → 짧은 호스트명 (예: srv01.domain.com → srv01)
_AR_HOST="${HOSTNAME:-$(hostname 2>/dev/null)}"
_AR_HOST="${_AR_HOST%%.*}"

# [개선] awk 의존성 제거: 파라미터 확장으로 첫 번째 필드(클라이언트 IP) 추출
_AR_IP="local"
[[ -n "${SSH_CONNECTION:-}" ]] && _AR_IP="${SSH_CONNECTION%% *}"

_AR_TTY_NAME="${_AR_TTY#/dev/}"
_AR_TTY_NAME="${_AR_TTY_NAME//\//_}"
_AR_FILE="${_AR_UDIR}/${_AR_TS}_${_AR_HOST}_${_AR_IP}_${_AR_TTY_NAME}.cast"

# ---------------------------------------------------------------------------
# 디렉터리 설정
# [BUG-2] 수정: 비루트 사용자는 chown root:USER 불가 — EUID 분기로 명시 처리
#
# 완전한 감사 무결성(사용자의 USER_DIR 삭제 방지)이 필요하다면
# root cron 또는 PAM pam_exec 모듈로 사전에 다음을 실행하도록 권장:
#   mkdir -p /var/log/asciinema/<username>
#   chown root:<username> /var/log/asciinema/<username>
#   chmod 770 /var/log/asciinema/<username>
# ---------------------------------------------------------------------------
_AR_EUID="${EUID:-$(id -u 2>/dev/null)}"

if [[ ! -d "${_AR_ROOT}" ]]; then
    if ! mkdir -p "${_AR_ROOT}" 2>/dev/null; then
        # 루트 디렉터리 생성 실패 → 녹화 불가, 세션은 정상 진입
        unset _AR_BIN _AR_TTY _AR_ROOT _AR_USER _AR_UDIR _AR_TS _AR_HOST _AR_IP _AR_TTY_NAME _AR_FILE _AR_EUID
        return 0 2>/dev/null
    fi
    if [[ "${_AR_EUID}" -eq 0 ]]; then
        chown root:root "${_AR_ROOT}"
        chmod 1777 "${_AR_ROOT}"
    fi
    # 비루트로 최초 실행: chmod 1777 불가. 관리자가 수동으로 사전 생성 필요.
fi

if [[ ! -d "${_AR_UDIR}" ]]; then
    if ! mkdir -p "${_AR_UDIR}" 2>/dev/null; then
        unset _AR_BIN _AR_TTY _AR_ROOT _AR_USER _AR_UDIR _AR_TS _AR_HOST _AR_IP _AR_TTY_NAME _AR_FILE _AR_EUID
        return 0 2>/dev/null
    fi

    if [[ "${_AR_EUID}" -eq 0 ]]; then
        # root 로그인: 타인 접근 완전 차단
        chown root:root "${_AR_UDIR}"
        chmod 700 "${_AR_UDIR}"
    else
        # 비루트: chown root:USER 는 root 권한 필요 — 수행 불가.
        # 사전 생성이 없으면 사용자 소유로 유지 (타인 읽기만 차단).
        chmod 700 "${_AR_UDIR}"
    fi
fi

# ---------------------------------------------------------------------------
# 보안 경고 배너
# ---------------------------------------------------------------------------
clear
printf '\033[1;31m\n'
printf '===============================================================\n'
printf '                [ SECURITY WARNING & NOTICE ]\n'
printf '===============================================================\n'
printf '\033[0m\n'
printf '본 서버는 내부 보안 정책에 따라 \033[1;33m모든 터미널 작업이 실시간 녹화\033[0m됩니다.\n'
printf '접속 IP   : %s\n' "${_AR_IP}"
printf '로그 경로 : %s\n' "${_AR_FILE}"
printf '\n'
printf '모든 명령어 입력 및 출력 결과는 감사 로그로 보관되며,\n'
printf '비인가된 활동은 관련 법규에 따라 처벌받을 수 있습니다.\n'
printf '\033[1;31m\n'
printf '===============================================================\n'
printf '\033[0m\n'
sleep 1

# ---------------------------------------------------------------------------
# 녹화 실행
# ---------------------------------------------------------------------------
export ASCIINEMA_REC=1
umask 022

_AR_SHELL="${SHELL:-/bin/bash}"
[[ -x "${_AR_SHELL}" ]] || _AR_SHELL="/bin/bash"

# [BUG-1] 핵심 수정: '-l' 플래그 제거
# 기존: --command "${LOGIN_SHELL} -l"
#   → 로그인 셸로 재기동 → /etc/profile → 이 스크립트 재소스 → asciinema 이중 실행
# 수정: '-l' 제거. exec 로 현재 셸을 대체하므로 로그인 환경(PATH, 환경변수 등)은
#   이미 상속됨. 비로그인 대화형 셸로도 동일한 환경이 제공된다.
exec "${_AR_BIN}" rec \
    --quiet \
    --idle-time-limit 2 \
    --command "${_AR_SHELL}" \
    "${_AR_FILE}"

# exec 실패 시 여기까지 도달 → 녹화 없이 세션 정상 진입 (사용자 로그인 차단 방지)
unset _AR_BIN _AR_TTY _AR_ROOT _AR_USER _AR_UDIR _AR_TS _AR_HOST _AR_IP _AR_TTY_NAME _AR_FILE _AR_EUID _AR_SHELL

# 1) 로그 저장소 초기화
sudo mkdir -p /var/log/asciinema
sudo chown root:root /var/log/asciinema
sudo chmod 1777 /var/log/asciinema

# 2. 크론 작업 내용을 파일로 생성 (root 권한 필요)
sudo tee /etc/cron.d/asciinema-cleanup <<'EOF'
# 매시간 정각마다 실행하여 90일이 지난 asciinema 로그(.cast) 삭제
# 매시간 0분에 실행 (00:00, 01:00, ...) / 90일 초과 cast 파일 삭제
0 * * * * root /usr/bin/find /var/log/asciinema -type f -name "*.cast" -mtime +90 -delete
EOF

# 3. 파일 권한 설정 (보안상 644 권한 권장)
sudo chmod 644 /etc/cron.d/asciinema-cleanup
```

## 참고 자료

- [Asciinema 공식 문서](https://docs.asciinema.org/getting-started/)

---
title: [도구명] 명령어 사용법
author: G.G
date: YYYY-MM-DD HH:MM +0900
categories: [Blog, Command]
tags: [Command, 태그1, 태그2]
---

## 📘 개요 (Overview)

[도구/명령어에 대한 1~3문장 설명]
- 어떤 문제를 해결하는지
- 어떤 환경에서 쓰이는지

## 📋 주요 옵션 (Options)

| 옵션 | 설명 | 예시 |
|------|------|------|
| `-x` | ... | `command -x value` |
| `--verbose` | 자세한 출력 | `command --verbose` |

> 옵션이 적거나 inline 설명으로 충분하면 이 섹션 생략 가능
{: .prompt-tip }

## ⚙️ 명령어 사용법

### [작업 목적 1]

```bash
# 설명 주석
command [options] <argument>
```

```
# 예상 출력 (필요 시)
output here
```
{: .prompt-info }

### [작업 목적 2]

```bash
# 설명 주석
command [options]
```

### [스크립트 예시] (해당하는 경우)

```bash
#!/usr/bin/env bash
set -euo pipefail

# 변수 정의
VAR="value"

# 실행
command "$VAR"
```

## ⚠️ 주의 사항 (Notes)

> 주의 사항이나 자주 발생하는 오류, 트러블슈팅이 있으면 작성
{: .prompt-warning }

## 참고 자료

- [공식 문서](URL)
- [관련 문서](URL)

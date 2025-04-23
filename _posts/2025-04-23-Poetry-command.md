---
title: Poetry 사용법
author: G.G
date: 2025-04-23 21:26 +0900
categories: [Blog, Command]
tags: [Command, Script, Poetry, Python3]
---

## 명령어 사용법

### 1️⃣ Poetry 설치 및 📌 설치 확인

```bash
# Poetry 공식 설치 스크립트 (macOS, Linux, Windows WSL 공통)
curl -sSL https://install.python-poetry.org | python3 -

# 설치 후 poetry가 실행되는지 확인
poetry --version
```

### 2️⃣ 새 프로젝트 생성

```bash
poetry new myprojects
cd myprojects
```

### 3️⃣ 패키지 의존성 추가 📦

```bash
poetry add pandas yfinance ta
```

### 4️⃣ 실행 (개발 환경 또는 CI/CD 자동 실행 시) 🧪

```bash
poetry run python myprojects/main.py
```

### 5️⃣ 패키지 목록 requirements.txt로 내보내기 📤

```bash
poetry export -f requirements.txt --output requirements.txt
```

```bash
[tool.poetry]
name = "myprojects"
version = "0.1.0"
description = "Stock data analysis with pandas + yfinance + ta"
authors = ["Your Name <you@example.com>"]

[tool.poetry.dependencies]
python = "^3.10"
pandas = "^2.2.1"
yfinance = "^0.2.37"
ta = "^0.11.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```
>
{: .prompt-info }
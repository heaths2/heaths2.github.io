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
# pipx 설치 방법 (Python3 기반 시스템)
python3 -m pip install --user pipx

# poetry 설치 (pipx 기반)
pipx install poetry

# 설치 후 pipx가 실행되는지 확인
pipx --version

# 설치 후 poetry가 실행되는지 확인
poetry --version
```

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
[project]
name = "myprojects"
version = "0.1.0"
description = ""
authors = [
    {name = "Your Name",email = "you@example.com"}
]
readme = "README.md"
requires-python = ">=3.12,<3.15"
dependencies = [
    "requests (>=2.32.3,<3.0.0)",
    "tabulate (>=0.9.0,<0.10.0)",
    "python-dotenv (>=1.1.0,<2.0.0)"
]

[tool.poetry]
packages = [{include = "myprojects", from = "src"}]


[tool.poetry.group.dev.dependencies]
black = "^25.1.0"
isort = "^6.0.1"
pytest = "^8.3.5"

[build-system]
requires = ["poetry-core>=2.0.0,<3.0.0"]
build-backend = "poetry.core.masonry.api"
```

> - pyproject.toml 설정 파일
{: .prompt-info }


## WSL + Poetry 프로젝트 초기화 스크립트

```bash
# ================================================
# ✅ 1. Poetry 설치 (Python 패키지/가상환경 관리 도구)
# ================================================
curl -sSL https://install.python-poetry.org | python3 -

# Poetry CLI를 사용할 수 있도록 PATH 설정
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# ================================================
# ✅ 2. 새 Python 프로젝트 생성
# ================================================
poetry new myprojects           # src/myprojects 구조 자동 생성
cd myprojects

# ================================================
# ✅ 3. Python 3.12 버전으로 가상환경 지정
# ================================================
# ※ WSL에 python3.12이 설치되어 있어야 함
poetry env use python3.12

# ================================================
# ✅ 4. 필수 라이브러리 설치 (프로덕션용)
# ================================================
poetry add requests             # ECOS API 호출용
poetry add tabulate             # 콘솔 표 출력
poetry add python-dotenv        # .env로 API 키 관리

# ================================================
# ✅ 5. 개발 도구 설치 (개발용 의존성)
# ================================================
poetry add --dev black isort pytest   # 코드 정리, 테스트용

# ================================================
# ✅ 6. 메인 스크립트 실행
# ================================================
# ※ src/myprojects/main.py 파일이 있어야 실행 가능
poetry run python src/myprojects/main.py
```

## 참고 자료
- [공식 문서서](https://python-poetry.org/docs/#installing-with-the-official-installer)
---
title: Hyper-V 설치 방법
author: G.G
date: 2025-06-16 21:32 +0900
categories: [Blog, Virtual Machine]
tags: [Virtual Machine, Hypervisor, Hyper-V]
---

## 📘 개요 (Overview)
이 문서는 Windows Home 에디션에서 Hyper-V 기능을 수동으로 설치하고 활성화하는 방법에 대해 안내합니다.
Hyper-V는 Windows Pro 이상에서만 공식 지원되며, Windows Home에는 기본적으로 제공되지 않습니다. 그러나 시스템 내부에는 Hyper-V 관련 패키지가 포함되어 있어, 이를 활용하면 제한적으로 Hyper-V 기능을 사용할 수 있습니다.

## ⚙️ 사용 방법

```bat
pushd "%~dp0"
dir /b %SystemRoot%\servicing\Packages\*Hyper-V*.mum >hyper-v.txt
for /f %%i in ('findstr /i . hyper-v.txt 2^>nul') do dism /online /norestart /add-package:"%SystemRoot%\servicing\Packages\%%i"
del hyper-v.txt
Dism /online /enable-feature /featurename:Microsoft-Hyper-V -All /LimitAccess /ALL
pause
```
{: file='Hyper-V.bat'}
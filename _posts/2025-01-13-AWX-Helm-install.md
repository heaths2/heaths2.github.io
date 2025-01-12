---
title: Helm을 이용한 AWX 설치
author: G.G
date: 2025-01-13 07:51:00 +0900
categories: [Blog, Orchestration]
tags: [kubernetes, k8s]
---

# Helm 명령어 정리

| **명령어**                  | **설명**                                                                                     |
|-----------------------------|---------------------------------------------------------------------------------------------|
| `helm repo add`            | Helm 저장소를 추가합니다.                                                                   |
| `helm repo update`         | Helm 저장소를 업데이트합니다.                                                               |
| `helm repo list`           | 추가된 Helm 저장소를 확인합니다.                                                             |
| `helm search repo`         | 추가된 저장소에서 차트를 검색합니다.                                                         |
| `helm search hub`          | Artifact Hub에서 차트를 검색합니다.                                                          |
| `helm install`             | 차트를 설치합니다.                                                                           |
| `helm upgrade`             | 이미 설치된 릴리스를 업그레이드합니다.                                                      |
| `helm rollback`            | 릴리스를 이전 버전으로 롤백합니다.                                                          |
| `helm uninstall`           | 릴리스를 제거합니다.                                                                         |
| `helm list`                | 현재 설치된 릴리스를 나열합니다.                                                             |
| `helm status`              | 특정 릴리스의 상태를 확인합니다.                                                             |
| `helm show chart`          | 차트의 정보를 표시합니다.                                                                    |
| `helm show values`         | 차트의 기본 값을 확인합니다.                                                                 |
| `helm show readme`         | 차트의 README 파일 내용을 확인합니다.                                                        |
| `helm template`            | Helm 차트를 템플릿으로 렌더링합니다.                                                         |
| `helm package`             | 차트를 `.tgz` 파일로 패키징합니다.                                                          |
| `helm create`              | 새 Helm 차트를 생성합니다.                                                                   |
| `helm lint`                | 차트를 검사하여 유효성을 검증합니다.                                                         |
| `helm get values`          | 특정 릴리스에 사용된 값을 확인합니다.                                                        |
| `helm get manifest`        | 특정 릴리스의 매니페스트를 확인합니다.                                                       |
| `helm get notes`           | 릴리스 설치 시 출력되는 메모를 확인합니다.                                                   |
| `helm get all`             | 릴리스와 관련된 모든 정보를 확인합니다.                                                      |
| `helm history`             | 특정 릴리스의 히스토리를 확인합니다.                                                        |
| `helm env`                 | Helm의 환경 변수를 확인합니다.                                                               |
| `helm dependency update`   | 차트의 종속성을 다운로드 및 업데이트합니다.                                                  |
| `helm dependency list`     | 차트의 종속성을 나열합니다.                                                                  |
| `helm dependency build`    | 차트의 종속성을 빌드합니다.                                                                  |
| `helm plugin list`         | 설치된 Helm 플러그인을 나열합니다.                                                           |
| `helm plugin install`      | 새로운 Helm 플러그인을 설치합니다.                                                          |
| `helm plugin update`       | 기존 Helm 플러그인을 업데이트합니다.                                                        |
| `helm plugin uninstall`    | 설치된 Helm 플러그인을 제거합니다.                                                           |

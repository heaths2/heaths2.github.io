---
title: Translate a Docker Compose File to Kubernetes Resources
author: G.G
date: 2025-08-19 11:41 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Kubernetes, Kompose]
---

## 📘 개요 (Overview)
Kompose는 Docker Compose 파일을 Kubernetes 리소스 매니페스트로 자동 변환해주는 오픈 소스 도구입니다. 이 도구를 사용하면 Docker 환경에서 개발된 애플리케이션을 Kubernetes 클러스터에 손쉽게 배포할 수 있습니다.

## 📌 목적 (Purpose)
- 🧭 등장배경: Docker Compose로 개발된 애플리케이션을 Kubernetes 환경으로 확장하려는 요구가 증가했습니다. Docker와 Kubernetes 간의 설정 및 배포 방식 차이로 인해 수동 변환 시 많은 시간과 노력이 필요했습니다.
- 🎯 기대효과: Kompose는 기존 Docker Compose 파일을 활용하여 Deployment, Service 등 복잡한 Kubernetes YAML 파일을 자동으로 생성합니다. 이를 통해 수동 작업으로 인한 오류를 줄이고, 개발자가 Kubernetes 환경에 빠르게 적응하여 생산성을 높일 수 있습니다.

## 📝 구성 요소 (Components)

| 역할 | 설명 | 특징 |
|---|---|---|
| **Kompose CLI** | Docker Compose 파일을 Kubernetes 리소스로 변환하는 핵심 명령어 도구입니다. | 단일 명령어(`kompose convert`)로 변환을 수행하며, 여러 Docker Compose 파일을 통합하여 변환할 수 있습니다. |
| **Docker Compose 파일** | 서비스, 네트워크, 볼륨 등을 정의하는 입력 파일입니다. | Kompose 변환의 기본 소스이며, `.yml` 또는 `.yaml` 확장자를 가집니다. |
| **Kubernetes 매니페스트 파일** | Kompose 변환 결과로 생성되는 출력 파일입니다. | `Deployment`, `Service`, `StatefulSet` 등 여러 종류의 리소스 파일을 포함하며, `kubectl apply` 명령어로 즉시 배포 가능합니다. |
| **컨테이너 이미지** | 애플리케이션 실행을 위해 필요한 기본 이미지입니다. | Docker Compose 파일에 정의된 이미지를 그대로 사용하여 Kubernetes 환경에 배포합니다. |

## ⚙️ 설지방법

- **Kompose 설치**

```bash
# Linux Kompose 바이너리 다운로드
curl -L https://github.com/kubernetes/kompose/releases/download/v1.26.0/kompose-linux-amd64 -o kompose

# 실행 권한 부여
chmod +x kompose

# 시스템 경로로 이동
sudo mv -v ./kompose /usr/local/bin/kompose
```

- **`docker-compose.yml` 파일 변환**

```bash
# 기본 변환: 현재 디렉터리 내의 docker-compose.yml 파일 단독 변환.
kompose convert

# 다중 파일 변환: -f 옵션 사용을 통한 여러 Docker Compose 파일 동시 변환.
kompose convert -f docker-compose.yml -f docker-guestbook.yml -o k8s
kompose convert -f docker-compose.yml -o k8s
# 또는
kompose convert --file docker-compose.yml --file docker-guestbook.yml  --out k8s
```

- **쿠버네티스 리소스 파일 적용**

```bash
# 쿠버네티스 리소스 파일 적용
kubectl apply -f nodejs-app-deployment.yaml
```

## 참고 자료
- [Kompose 공식 문서](https://kubernetes.io/ko/docs/tasks/configure-pod-container/translate-compose-kubernetes/)

---
title: OpenStack 설치 매뉴얼
author: G.G
date: 2025-01-31 09:27:00 +0900
categories: [Blog, Virtual Machine]
tags: [provisioning, openstack, ansible, docker]
---

## 개요
Kolla-Ansible은 OpenStack을 컨테이너 기반으로 배포하고 관리하기 위한 도구다. Kolla 프로젝트는 Docker 컨테이너와 Ansible을 결합하여 OpenStack의 설치, 구성 및 유지보수를 자동화한다. 이를 통해 복잡한 OpenStack 환경을 보다 쉽게 운영할 수 있다.

## 특징
1. 컨테이너 기반 배포: Docker 컨테이너를 활용하여 OpenStack 서비스를 분리 및 관리
2. Ansible 자동화: Ansible Playbook을 통해 설치 및 업그레이드 자동화
3. 유연한 구성: 다양한 환경에 맞춰 배포 가능 (AIO, 멀티 노드, HA 지원)
4. 빠른 복구 및 롤백: 컨테이너를 이용한 손쉬운 서비스 재시작 및 롤백 가능
5. 보안 강화: 구성 가능한 TLS 및 인증서 관리 지원

## 구성 요소
1. Kolla: OpenStack 서비스의 Docker 컨테이너 이미지를 제공하는 프로젝트
2. Ansible Playbooks: OpenStack 배포 및 유지보수를 자동화하는 Ansible 스크립트
3. Kolla-Ansible CLI: 배포 및 유지보수 작업을 수행하는 명령줄 인터페이스
4. Inventory 파일: 배포할 노드 및 역할을 정의하는 설정 파일
5. Global Configuration (globals.yml): OpenStack의 전반적인 설정을 정의하는 파일
6. Container Registry: OpenStack 컨테이너 이미지를 저장하는 저장소 (Docker Hub, 프라이빗 레지스트리 등)

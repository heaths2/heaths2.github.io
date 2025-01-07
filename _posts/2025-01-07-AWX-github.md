---
title: AWX Github 연동
author: G.G
date: 2025-01-07 21:08:00 +0900
categories: [Blog, Orchestration]
tags: [ansible, awx, github]
---

> 관련글 :
> [ Kubespray 설치방법_(1)](https://heaths2.github.io/posts/kubespray_install/)
> [ Ansible AWX 설치방법_(2)](https://heaths2.github.io/posts/AWX-install/)

## 개요
AWX는 Ansible Tower의 오픈 소스 프로젝트로, 자동화된 작업 스케줄링 및 관리에 유용합니다. AWX는 외부 버전 관리 시스템(VCS)과 통합하여 Playbook 및 프로젝트 파일을 관리할 수 있습니다. 이 매뉴얼에서는 GitHub를 AWX와 연동하여 프로젝트를 관리하는 방법에 대해 설명합니다.
GitHub 연동을 통해 다음과 같은 작업이 가능합니다.
- GitHub에서 Playbook 및 프로젝트를 자동으로 가져오기.
- GitHub의 변경 사항을 AWX에 자동으로 동기화.
- GitHub의 브랜치 또는 태그별로 Playbook 실행.

## 특징
1. 중앙화된 프로젝트 관리
- GitHub와 연동하여 중앙에서 Playbook을 관리.
- Git의 브랜치 관리 기능을 활용하여 다양한 작업 환경(개발, 테스트, 프로덕션)을 분리.

2. 자동 동기화  
- GitHub 저장소에 변경 사항이 발생하면 AWX에서 자동으로 업데이트.
- Webhook을 사용하여 GitHub의 변경 사항을 실시간 반영 가능.

3. 보안 관리
- HTTPS 또는 SSH 인증 방식을 지원하여 보안 강화.
- GitHub 개인 액세스 토큰 또는 SSH 키를 사용해 인증.

3. 다양한 브랜치 및 태그 선택
- 특정 브랜치 또는 태그를 선택하여 AWX에서 실행.

## 환경설정

### SSH 키 생성 및 등록
1. SSH 키 생성

```bash
echo "yes" | ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa -C Github
```

2. SSH 공개 키 확인 및 복사

```bash
cat /root/.ssh/id_rsa.pub
```

3. GitHub의 [SSH and GPG keys](https://github.com/settings/keys) 페이지로 이동

![AWX-Github_1](/assets/img/2025-01-07/AWX-Github_1.jpg)
_Github SSH and GPG keys_

4. Add SSH key 버튼을 클릭하여 저장

![AWX-Github_2](/assets/img/2025-01-07/AWX-Github_2.jpg)
__Github Add SSH key_

5. 연결 테스트

```bash
ssh -T git@github.com
Hi infra! You've successfully authenticated, but GitHub does not provide shell access.
```

## 

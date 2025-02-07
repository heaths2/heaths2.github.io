---
title: AWX Github 연동
author: G.G
date: 2025-01-07 21:08:00 +0900
categories: [Blog, Orchestration]
tags: [ansible, awx, github]
---

> 관련글 :
> - [ Kubespray 설치방법_(1)](https://heaths2.github.io/posts/kubespray_install/)
> - [ Ansible AWX 설치방법_(2)](https://heaths2.github.io/posts/AWX-install/)

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

![AWX-Github_1](/assets/img/2025-01-07/AWX-Github_1.png)
_Github SSH and GPG keys_

4. Add SSH key 버튼을 클릭하여 저장

![AWX-Github_2](/assets/img/2025-01-07/AWX-Github_2.png)
__Github Add SSH key_

5. 연결 테스트

```bash
ssh -T git@github.com
Hi infra! You've successfully authenticated, but GitHub does not provide shell access.
```

## AWX Github 연동

### AWX에서 자격 증명 추가
1. 인증 정보: AWX에서 인증 정보 메뉴로 이동.
2. 추가: 새로운 인증 정보를 추가하기 위해 버튼 클릭.

![AWX-Github_3](/assets/img/2025-01-07/AWX-Github_3.jpg)
__Github 인증정보 추가_

### 새 인증 정보 생성
1. 이름: 인증 정보를 구별할 이름 입력 (예: "Github SSH-KEY").
2. 인증 정보 유형: "소스 제어"를 선택.
3. SSH 키: 생성한 개인 키를 입력.
4. 저장: 입력을 완료하고 저장 버튼 클릭.

![AWX-Github_4](/assets/img/2025-01-07/AWX-Github_4.jpg)
__Github 인증정보 생성_

### 프로젝트 추가
1. 프로젝트: AWX에서 프로젝트 메뉴로 이동.
2. 추가: 새로운 프로젝트 추가를 위해 버튼 클릭.

![AWX-Github_5](/assets/img/2025-01-07/AWX-Github_5.jpg)
__Github 프로젝트 추가_

### 새 프로젝트 생성
1. 이름: 프로젝트 이름을 입력 (예: "Ansible-AWX")
2. 소스 제어 유형: Git을 선택.
3. 소스 제어 URL: 연결할 GitHub 저장소의 URL 입력 (예: git@github.com:infra/Ansible-AWX.git).
4. 소스 제어 인증 정보: 이전에 생성한 인증 정보(예: "Github SSH-KEY")를 선택.
5. 옵션:
  - [x] 정리: 기존의 불필요한 파일 제거.
  - [ ] 삭제: 현재 저장소를 삭제 후 다시 클론.
  - [ ] 하위 모듈 추적: Git 서브모듈 사용 시 체크.
  - [x] 시작 시 버전 업데이트: 시작 시 저장소 업데이트 수행.
6. 소스 제어 분기/태그 설정 (선택 사항)
- 소스 제어 분기/태그/커밋: 특정 분기(예: "main"), 태그 또는 커밋을 지정(선택 사항)
![AWX-Github_6](/assets/img/2025-01-07/AWX-Github_6.jpg)
__Github 프로젝트 생성_

### 소스 제어 인증 정보 연결
1. 인증 정보: 프로젝트에서 사용할 인증 정보 선택 (예: "Github SSH-KEY").
2. 선택: 원하는 인증 정보를 선택하고 확인.

![AWX-Github_7](/assets/img/2025-01-07/AWX-Github_7.jpg)
__Github 소스 제어 인증 정보 연결_

### 프로젝트 성공 확인
프로젝트 상태: 프로젝트 상태가 성공으로 표시되면 GitHub 연동 완료.

![AWX-Github_8](/assets/img/2025-01-07/AWX-Github_8.jpg)
__Github 연동 동기화_

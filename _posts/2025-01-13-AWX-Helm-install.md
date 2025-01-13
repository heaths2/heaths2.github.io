---
title: Helm을 이용한 AWX 설치
author: G.G
date: 2025-01-13 07:51:00 +0900
categories: [Blog, Orchestration]
tags: [kubernetes, k8s]
---

> 관련글 :
> [ Kubespray 설치방법 (1)](https://heaths2.github.io/posts/kubespray_install/)
> [ Ansible AWX 설치방법 (2)](https://heaths2.github.io/posts/AWX-install/)
> 

## 개요
AWX는 Ansible의 웹 기반 관리 툴로, Ansible 작업을 자동화하고 관리할 수 있는 강력한 기능을 제공합니다. Helm 차트를 사용하면 Kubernetes 클러스터에 AWX를 쉽고 일관되게 배포할 수 있습니다.

## 환경구성
이 작업은 외부 PostgreSQL 데이터베이스 서버를 설정하고, 기존 AWX 데이터를 백업 파일을 통해 복원하는 과정을 포함합니다.

### AWX 데이터베이스 백업

```bash
kubectl exec -it awx-server-postgres-15-0 -- pg_dump -U awx -w -Fp -b -v -f /tmp/awx.sql -d awx
```

### AWX 데이터베이스 백업 파일 로컬로 복사

```bash
kubectl cp -n awx awx-server-postgres-15-0:/tmp/awx.sql ~/awx/awx.sql
```

### 새 사용자 및 데이터베이스 생성 스크립트 생성

```bash
cat <<EOF > /tmp/awx-generation.sql
-- awx 사용자 생성 (비밀번호는 원하는 대로 설정)
CREATE USER awx WITH PASSWORD 'awx';

-- awx 데이터베이스 생성 및 소유권 부여
CREATE DATABASE awx OWNER awx;

-- awx 사용자에게 데이터베이스에 대한 모든 권한 부여
GRANT ALL PRIVILEGES ON DATABASE awx TO awx;
EOF
```

### 데이터베이스 및 사용자 생성 PostgreSQL 스크립트 실행

```bash
sudo -u postgres psql -f /tmp/awx-generation.sql
```

### AWX 테이블 및 데이터 생성 PostgreSQL 스크립트 실행

```bash
sudo -u postgres psql -f /tmp/awx.sql
```

### 마이그레이션 적용

```bash
kubectl exec -it deployment/awx-web -n awx -- awx-manage migrate
```

### 마이그레이션 상태 확인

```
kubectl exec -it deployment/awx-web -n awx -- awx-manage showmigrations
```

### `admin` 사용자 비밀번호 변경

```bash
kubectl exec -it deployment/awx-web -n awx -- awx-manage changepassword admin
```

### PostgreSQL 데이터베이스 테이블 및 사용자 목록 확인

```bash
sudo -u postgres psql -d awx -c "\dt"
sudo -u postgres psql -c "\du"
```

## Helm AWX Operator 설치 매뉴얼

### Helm 명령어어 정의
1. Helm 명령어 정리

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

2. Helm 차트의 레이아웃 구성요소

| **경로/파일명**                  | **설명**                                                                                     |
|-----------------------------|---------------------------------------------------------------------------------------------|
| `Chart.yaml`                | 차트에 대한 메타데이터를 포함합니다. (예: 이름, 버전, 설명 등)                                  |
| `values.yaml`               | 차트의 기본 값을 정의합니다. 배포 시 사용자가 값을 덮어쓸 수 있습니다.                            |
| `templates/`                | Kubernetes 리소스 템플릿이 포함된 디렉터리입니다.                                                |
| `templates/deployment.yaml` | 애플리케이션 Pod 관리를 위한 Deployment 리소스 템플릿입니다.                                     |
| `templates/service.yaml`    | 애플리케이션을 외부로 노출하기 위한 Service 리소스 템플릿입니다.                                 |
| `templates/ingress.yaml`    | 도메인 기반 라우팅을 위한 Ingress 리소스 템플릿입니다.                                           |
| `templates/NOTES.txt`       | 차트 설치 후 표시되는 메모(설치 방법 및 사용법 등)를 포함합니다.                                 |
| `templates/_helpers.tpl`    | 재사용 가능한 템플릿 헬퍼 함수 또는 변수를 정의합니다.                                           |
| `charts/`                   | 서브차트 디렉터리로, 차트의 의존성을 관리합니다.                                                |
| `crds/`                     | 사용자 정의 리소스 정의(CRDs)가 포함된 디렉터리입니다.                                           |
| `tests/`                    | 차트의 기능을 테스트하기 위한 스크립트를 포함합니다.                                             |

3. Helm 차트의 레이아웃 Tree 구조

```bash
my-chart/
├── Chart.yaml                # 차트에 대한 메타데이터
├── values.yaml               # 기본 변수 값 정의
├── templates/                # Kubernetes 리소스 템플릿
│   ├── deployment.yaml       # Deployment 리소스 템플릿
│   ├── service.yaml          # Service 리소스 템플릿
│   ├── ingress.yaml          # Ingress 리소스 템플릿
│   ├── NOTES.txt             # 설치 후 표시되는 메모
│   ├── _helpers.tpl          # 재사용 가능한 템플릿 함수 정의
├── charts/                   # 서브차트 디렉터리
├── crds/                     # 사용자 정의 리소스 정의(CRDs)
└── tests/                    # 차트 테스트 스크립트
```

### Helm 설치치

### Helm Bash 자동 완성 활성화

```bash
source <(helm completion bash)
echo "source <(helm completion bash)" >> ~/.bashrc
```

## 참조
---
title: OpenStack 설치 매뉴얼
author: G.G
date: 2025-01-31 09:27:00 +0900
categories: [Blog, Provisioning]
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

## 환경 구성
### 사전 준비 
nscd(Name Service Cache Daemon) 비활성화 (DNS 캐시 충돌 방지)

```bash
systemctl disable --now nscd.service
systemctl mask nscd.service
```

### NFS 설정

1. 스토리지 노드에서 LVM 볼륨 그룹 및 논리 볼륨 생성
```bash
vgcreate Volumes /dev/sdb
lvcreate -l 100%FREE -n cinder-volumes Volumes
```

2. 마운트 및 fstab 설정

```bash
mkfs.ext4 /dev/Volumes/cinder-volumes
mkdir -pv /data

cat <<EOF >> /etc/fstab
# Storage node
/dev/Volumes/cinder-volumes /data ext4 defaults 0 0
EOF

mount -a
```

3. NFS 서버 설치 및 설정
```bash
apt install -y nfs-kernel-server
mkdir -pv /data/cinder-volumes
```

4. NFS 익스포트 설정(/etc/exports)
```bash
cat <<EOF >> /etc/exports
/data/cinder-volumes 10.1.1.101/32(rw,sync,no_subtree_check,no_root_squash)
EOF
```

5. NFS 서비스 적용
```bash
NFS 서비스 적용
exportfs -rv
showmount -e 10.1.1.104
```

6. Control node NFS 마운트 설정 (/etc/fstab)

```bash
cat <<EOF >> /etc/fstab

# Storage node
10.1.1.104:/data/cinder-volumes /data/cinder-volumes nfs4 rw,relatime,vers=4.2,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,nofail 0 0
EOF
mkdir -pv /data/cinder-volumes
mount -a
```

## 설치
### Kolla-Ansible 설치

1. 필수 패키지 설치
```bash
sudo apt update
sudo apt install -y git python3-dev python3-pip python3-venv libffi-dev gcc libssl-dev
```

2. Python 가상 환경 설정
```bash
cd /opt
python3 -m venv kolla-ansible/venv
source kolla-ansible/venv/bin/activate
```

3. Kolla-Ansible 설치
```bash
pip install -U pip
pip install 'ansible-core>=2.16,<2.17.99'
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2024.2
```

4. 설정 디렉터리 생성 및 권한 변경
```bash
sudo mkdir -pv /etc/kolla
sudo chown -v $USER:$USER /etc/kolla
```

5. 기본 설정 파일 복사
```bash
cp -rv /opt/kolla-ansible/venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp -rv /opt/kolla-ansible/venv/share/kolla-ansible/ansible/inventory/* /etc/kolla
```

6. 필수 의존성 설치
```bash
kolla-ansible install-deps
```

### OpenStack 설정

1. 관리자 비밀번호 설정 (keystone_admin_password)
```bash
kolla-genpwd
sed -i 's/^keystone_admin_password: .*/keystone_admin_password: admin' /etc/kolla/passwords.yml
```

2. OpenStack 환경 변수 설정
```bash
sed -i -e 's/^#network_interface: .*/network_interface: "xenbr1"/' \
       -e 's/^#kolla_base_distro: .*/kolla_base_distro: "ubuntu"/' \
       -e 's/^#kolla_internal_vip_address: .*/kolla_internal_vip_address: "10.1.1.100"/' \
       -e 's/^#kolla_external_vip_interface: .*/kolla_external_vip_interface: "xenbr0"/' \
       -e 's/^#neutron_external_interface: .*/neutron_external_interface: "xenbr0"/' \
       -e 's/^#enable_cinder: .*/enable_cinder: "yes"/' \
       -e 's/^#enable_cinder_backend_nfs: .*/enable_cinder_backend_nfs: "yes"/' \
       -e 's/^#enable_neutron_dvr: .*/enable_neutron_dvr: "yes"/' \
       -e 's/^#enable_neutron_provider_networks: .*/enable_neutron_provider_networks: "yes"/' \
       -e 's/^#enable_skyline: .*/enable_skyline: "yes"/' \
       -e 's/^#nova_compute_virt_type: .*/nova_compute_virt_type: "qemu"/' \
       /etc/kolla/globals.yml
```

3. 설정 확인
```bash
grep -v "#" /etc/kolla/globals.yml | grep -v "^$"
```

### OpenStack 배포

1. 서버 기본 설정
```bash
kolla-ansible bootstrap-servers -i multinode
```

2. 설치 전 점검 (네트워크, 패키지, 설정 오류 확인)
```bash
kolla-ansible prechecks -i multinode
```

3. OpenStack 컨테이너 이미지 다운로드
```bash
kolla-ansible pull -i multinode
```

4. OpenStack 배포 실행
```bash
kolla-ansible deploy -i multinode
```

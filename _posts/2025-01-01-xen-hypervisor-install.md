---
title: XEN Hypervisor 설치방법
author: G.G
date: 2025-01-01 13:00:00 +0900
categories: [Blog, Virtual Machine]
tags: [virtual machine, xen, xen hypervisor]
---

## 개요
Xen Hypervisor는 고성능 및 고효율의 오픈소스 **하이퍼바이저(Hypervisor)**로, 가상화 환경에서 다중 운영체제를 단일 물리 서버에서 실행할 수 있도록 지원하는 핵심 소프트웨어입니다. Xen은 Type-1 하이퍼바이저로 분류되며, 하드웨어 위에서 직접 실행되면서 높은 성능과 보안을 제공합니다.

## 특징
1. Type-1 하이퍼바이저
- Xen은 하드웨어 위에서 직접 실행되는 Type-1 하이퍼바이저로, 게스트 운영체제와 독립적으로 작동하며 성능과 안정성이 뛰어납니다.

2. 오픈소스 기반
- Xen은 GNU General Public License(GPL)로 배포되는 오픈소스 프로젝트로, 커뮤니티와 상용 지원을 모두 받을 수 있습니다.

3. 가상화 방식 지원
- 전가상화(Paravirtualization): 게스트 운영체제를 수정하여 더 높은 성능 제공.
- 완전 가상화(Hardware-Assisted Virtualization): CPU 가상화 기술(Intel VT, AMD-V)을 활용하여 게스트 운영체제를 수정하지 않고도 가상화 가능.

4. 보안 중심 설계
- Xen은 다중 격리와 최소 신뢰 컴퓨팅 모델을 사용하여 높은 보안을 제공합니다.

5. 다양한 운영체제 지원
- Xen은 Linux, Windows, BSD, Solaris 등 다양한 게스트 운영체제를 지원합니다.

6. 클라우드 및 서버 환경 최적화
- Xen은 대규모 클라우드 환경과 데이터센터에서 최적화되어 있으며, **AWS, Citrix Hypervisor(구 XenServer)**와 같은 주요 클라우드 솔루션에서 사용됩니다.

## 구성 요소

1. Xen Hypervisor
- 물리 서버의 하드웨어와 직접 상호작용하며, 가상 머신(Virtual Machine, VM)을 생성 및 관리합니다.

2. Domain 0 (Dom0)
- Xen이 부팅 시 생성하는 특별한 가상 머신으로, 하드웨어에 대한 직접 접근 권한을 가지고 다른 VM을 관리합니다.

3. Domain U (DomU)
- 사용자가 생성한 일반적인 가상 머신으로, Dom0를 통해 관리됩니다.

## 환경구성
> HPE Proliant DL360 Gen9 서버를 활용하여 Ubuntu 24.04에서 환경 구성합니다.

### 네트워크 설정
1. NIC 설정

| **구분**    | **IP 대역**             | **VLAN**  | **용도**                                    |
|-------------|-------------------------|-----------|----------------------------------------------|
| 공인 IP 1   | 203.0.113.0/24          | VLAN 100  | 공인 IP 예시 1 (임의의 테스트용 IP 대역)      |
| 공인 IP 2   | 203.51.100.0/24         | VLAN 200  | 공인 IP 예시 2 (임의의 테스트용 IP 대역)      |
| 사설 IP 1   | 10.1.1.0/24             | VLAN 1000 | 사설 IP Domain-0 관리 IP                     |

```bash
tee /etc/netplan/50-cloud-init.yaml << EOF
# This is the network config written by 'subiquity'
network:
  ethernets:
    eno1:
      dhcp4: false
      dhcp6: false
    eno2:
      dhcp4: false
      dhcp6: false
    eno3:
      dhcp4: false
      dhcp6: false
    eno4:
      dhcp4: false
      dhcp6: false
  bonds:
    bond0:
      interfaces: [ eno1, eno2 ]
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
    bond1:
      interfaces: [ eno3, eno4 ]
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
  vlans:
    bond0.100:
      id: 100
      link: bond0
    bond0.200:
      id: 200
      link: bond0
  bridges:
    xenbr0:
      dhcp4: false
      dhcp6: false
      interfaces: [ bond0.100 ]
    xenbr1:
      dhcp4: false
      dhcp6: false
      interfaces: [ bond0.200 ]
    xenbr2:
      dhcp4: false
      dhcp6: false
      interfaces: [ bond0.1000 ]
      addresses: [ 10.1.1.100/24 ]
      nameservers:
        addresses: [ 168.126.63.1, 164.124.101.2, 210.220.163.82, 8.8.8.8 ]
        search: [ in.infra.co.kr ]
      routes:
        - to: default
          via: 10.1.255.1
  version: 2
EOF
```

2. 패키지 설치

```bash
sudo apt update
sudo apt install -y xen-hypervisor-4.17-amd64 xen-utils-4.17 xen-tools
```

3. 저장소 생성
- 기본적으로 /dev/sda가 OS영역으로 할당되어 있음

A. LVM 방식
- 디스크 파티션 생성

```bash
parted -s /dev/sdb mklabel gpt mkpart primary 0 100% print quit
# 또는
parted --script /dev/sdb mklabel gpt mkpart primary 0 100% print quit
```

- 볼륨 그룹 생성(물리 볼륨 생성 pvcreate /dev/sdb 자동으로 생성됨)

```bash
vgcreate Disks /dev/sdb
```

B. 디스크 방식
- 디스크 파티션 생성

```bash
parted -s /dev/sdb mklabel gpt mkpart primary 0 100% print quit
# 또는
parted --script /dev/sdb mklabel gpt mkpart primary 0 100% print quit
```

- 볼륨 그룹 생성(물리 볼륨 생성 pvcreate /dev/sdb 자동으로 생성됨)

```bash
vgcreate Disks /dev/sdb
```

- LVM 100% 할당

```bash
lvcreate -l 100%FREE -n img Disks
# 또는
lvcreate --extents 100%FREE -n img Disks
```

- ext4 파일 시스템 생성

```bash
mkfs.ext4 /dev/Disks/img
```

- 이미지 경로 마운트

```bash
mkdir -pv /data/volumes
mount /dev/Disks/img /data/volumes
```

- 부팅시 자동으로 마운트되도록 등록

```bash
echo '/dev/Disks/img /data/volumes ext4 defaults 0 0' >> /etc/fstab
```

4. 오류 발생시
- 오류 발생: `vgcreate Disks /dev/sdb`

```bash
vgcreate Disks /dev/sdb
  Device /dev/sdb excluded by a filter.
```

- 메타데이터 제거: `wipefs -a /dev/sdb`

```bash
wipefs -a /dev/sdb
/dev/sdb: 8 bytes were erased at offset 0x00000200 (gpt): 45 46 49 20 50 41 52 54
/dev/sdb: 8 bytes were erased at offset 0xb54ba25e00 (gpt): 45 46 49 20 50 41 52 54
/dev/sdb: 2 bytes were erased at offset 0x000001fe (PMBR): 55 aa
/dev/sdb: calling ioctl to re-read partition table: Success
```

- 물리 볼륨 생성: `pvcreate /dev/sdb`

```bash
pvcreate /dev/sdb
```

- 볼륨 그룹 생성: `vgcreate Disks /dev/sdb`

```bash
vgcreate Disks /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```

## XEN Hypervisor 설치

1. 
2. 3
3. 4
4. 5
5. 

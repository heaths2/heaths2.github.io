---
title: iPXE 설치방법
author: G.G
date: 2025-01-02 16:12:00 +0900
categories: [Blog, Provisioning]
tags: [provisioning, ipxe]
---

## 개요
iPXE는 오픈 소스 네트워크 부트 소프트웨어로, PXE(Preboot Execution Environment)를 확장하여 네트워크 기반 부팅을 더욱 유연하게 지원합니다. 디스크리스 부팅, 클라우드 환경 통합, 운영 체제 설치 및 대규모 시스템 프로비저닝에서 널리 사용됩니다. HTTP, HTTPS, iSCSI, TFTP 등 다양한 프로토콜을 지원하며, 스크립트 기능을 통해 복잡한 부팅 프로세스를 자동화할 수 있습니다.

## 특징
1. 다양한 프로토콜 지원
- TFTP, HTTP(S), iSCSI, FCoE, AoE 등 현대적인 네트워크 프로토콜을 지원.

2. 스크립트 기반 자동화
- 스크립트를 작성하여 조건부 부팅, 동적 네트워크 설정, 커널 선택 등을 수행.

3. 클라우드 및 베어 메탈 통합
- OpenStack, OpenShift, AWS, GCP 등과 연동하여 베어 메탈 및 클라우드 인프라에 OS 배포 가능.

4. BIOS 및 UEFI 지원
- 다양한 하드웨어 및 가상 환경에서 동작 가능.

5. 확장성과 유연성
- PXE의 한계를 극복하며 커스터마이징 가능.

## 구성 요소
1. iPXE 바이너리
- 네트워크 부팅을 실행하는 기본 프로그램.
- 다양한 형태로 배포 가능:
  - ISO 이미지
  - USB 또는 플로피 디스크
  - 네트워크에서 직접 다운로드.

2. 부트 로더
- BIOS/UEFI 환경에서 실행되어 네트워크 설정 및 이미지 로드 시작.

3. 스크립트
- iPXE의 동작을 제어하는 스크립트.
- 예: 네트워크 설정, 부팅 이미지 선택, 조건부 로직.

```bash
#!ipxe
dhcp
chain http://boot.ipxe.org/demo/boot.php
```

4. 네트워크 프로토콜 지원
- TFTP: 전통적인 PXE 부팅 방식.
- HTTP/HTTPS: 최신 프로토콜로, 빠르고 안전한 파일 전송 가능.
- iSCSI/NFS: 네트워크 스토리지에서 OS를 직접 부팅.

5. 동적 이미지 로딩
- 클라우드 환경에서 최신 OS 이미지를 동적으로 로드.

## 환경구성

### TFTP 서버 설정
1. TFTP 서버 설치
2. TFTP 서버 설정
3. TFTP 서버 서비스 재시작

```bash
sudo apt install tftpd-hpa
```

```bash
tee /etc/default/tftpd-hpa << EOF
# /etc/default/tftpd-hpa

TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure --create --list -vvv"
RUN_DAEMON="yes"
OPTIONS=" -l -s -r -blksize 1468 /srv/tftp"
EOF
```

```bash
systemctl restart tftpd-hpa.service
```

### DHCP 서버 설정
1. DHCP 서버 설치
2. DHCP 서버 설정
3. `/etc/dhcp/dhcpd.conf`{: .filepath} 파일 구문 및 유효성을 검사
4. TFTP 서버 서비스 재시작

```bash
sudo apt install isc-dhcp-server
```

```bash
IPADD=$(hostname -I | awk '{print $1}')
IPGW=$(ip route | grep default | head -n 1 | awk '{print $3}')
IPADD_PREFIX=$(echo $IPADD | sed 's/\.[0-9]*$//')

cat << EOF >> /etc/dhcp/dhcpd.conf
option space PXE;
option PXE.mtftp-ip    code 1 = ip-address;
option PXE.mtftp-cport code 2 = unsigned integer 16;
option PXE.mtftp-sport code 3 = unsigned integer 16;
option PXE.mtftp-tmout code 4 = unsigned integer 8;
option PXE.mtftp-delay code 5 = unsigned integer 8;
option arch code 93 = unsigned integer 16; # RFC4578

use-host-decl-names on;
ddns-update-style interim;
ignore client-updates;
next-server 10.1.81.3;
authoritative;

subnet 10.1.0.0 netmask 255.255.0.0 {
    option routers $IPGW;
    option subnet-mask 255.255.0.0;
    option domain-name-servers 1.1.1.1, 8.8.8.8;
    option time-offset 32400;     # Timezone: Asia/Seoul
    default-lease-time 600;
    max-lease-time 7200;
    range $IPADD_PREFIX.100 $IPADD_PREFIX.200;
    next-server 10.1.81.3;
 
    class "UEFI-32-1" {
    match if substring(option vendor-class-identifier, 0, 20) = "PXEClient:Arch:00006";
    filename "ipxe/bin/ipxe.efi";
    }

    class "UEFI-32-2" {
    match if substring(option vendor-class-identifier, 0, 20) = "PXEClient:Arch:00002";
     filename "ipxe/bin/ipxe.efi";
    }

    class "UEFI-64-1" {
    match if substring(option vendor-class-identifier, 0, 20) = "PXEClient:Arch:00007";
     filename "ipxe/bin/ipxe.efi";
    }

    class "UEFI-64-2" {
    match if substring(option vendor-class-identifier, 0, 20) = "PXEClient:Arch:00008";
    filename "ipxe/bin/ipxe.efi";
    }

    class "UEFI-64-3" {
    match if substring(option vendor-class-identifier, 0, 20) = "PXEClient:Arch:00009";
     filename "ipxe/bin/ipxe.efi";
    }

    class "Legacy" {
    match if substring(option vendor-class-identifier, 0, 20) = "PXEClient:Arch:00000";
    filename "ipxe/bin/undionly.kkpxe";
    }
}
EOF
```

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

```bash
systemctl restart isc-dhcp-server
```

### NFS 서버 설정
1. NFS 서버 설치
2. NFS 서버 설정
4. NFS 서버 서비스 재시작

```bash
sudo apt install nfs-kernel-server
```

```bash
cat <<EOF | sudo tee -a /etc/exports
/srv/tftp/ubuntu24.04   10.1.81.0/16(async,no_root_squash,no_subtree_check,ro)
/srv/tftp/ubuntu22.04   10.1.81.0/16(async,no_root_squash,no_subtree_check,ro)
/srv/tftp/ubuntu20.04   10.1.81.0/16(async,no_root_squash,no_subtree_check,ro)
EOF
```

```bash
exportfs -rv
```

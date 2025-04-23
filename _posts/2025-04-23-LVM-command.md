---
title: LVM(Logical Volume Management) 명령어 사용법
author: G.G
date: 2025-04-23 11:53:00 +0900
categories: [Blog, Command]
tags: [Command, Script, DMI, dmidecode]
---

## 명령어 사용법

### 📦 디스크/볼륨 확인

```bash
# 블록 디바이스 구조 확인 (디스크, 파티션, LVM 논리 볼륨 포함)
lsblk
```

```bash
NAME              MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                 8:0    0   50G  0 disk 
├─sda1              8:1    0    1G  0 part /boot
└─sda2              8:2    0   49G  0 part 
  ├─Disks_OS-root 253:0    0   37G  0 lvm  /
  ├─Disks_OS-swap 253:1    0    8G  0 lvm  [SWAP]
  └─Disks_OS-tmp  253:2    0    4G  0 lvm  /tmp
sdb                8:16    0    1T  0 disk
sdc                8:32    0   20T  0 disk
sdd                8:48    0  100T  0 disk 
sr0                11:0    1 1024M  0 rom  
```
{: .prompt-info }

### 🧱 LVM 볼륨 그룹 및 논리 볼륨 생성

```bash
# 새 볼륨 그룹 생성 (Disks_OS: /dev/sdb 사용)
vgcreate Disks_OS /dev/sdb

# 데이터용 볼륨 그룹 생성
vgcreate Disks_DATA /dev/sdc

# 로그용 볼륨 그룹 생성 (같은 디바이스 재사용했으므로 실제로는 덮어쓰기 주의)
vgcreate Disks_LOG /dev/sdc
```

### 🧱 논리 볼륨 생성 (LV)

```bash
# 운영 디스크: 50G
lvcreate -L 50G -n DEV-WWW-disk Disks_OS

# 데이터 디스크: 100G
lvcreate -L 100G -n DEV-WWW-data Disks_DATA

# 스왑 공간: 1G
lvcreate -L 1G -n DEV-WWW-swap Disks_OS

# 남은 전체 공간을 한 번에 할당 (두 방식 동일)
lvcreate -l 100%FREE -n /dev/Disks_LOG/DATA Disks_LOG
lvcreate --extents 100%FREE -n /dev/Disks_LOG/DATA Disks_LOG
```

### 🗃️ 파일 시스템 생성 및 스왑 설정

```bash
# ext4 파일 시스템 생성
mkfs.ext4 /dev/Disks_OS/DEV-WWW-disk
mkfs.ext4 /dev/Disks_DATA/DATA
mkfs.ext4 /dev/Disks_LOG/LOG

# 스왑 공간 초기화
mkswap /dev/Disks_OS/DEV-WWW-swap
```

### 🔄 논리 볼륨 삭제

```bash
# 특정 이름 패턴을 가진 LV 일괄 삭제
lvremove /dev/Disks_OS/DEV-WWW-*
```

### 🧱 논리 볼륨 확장 후 파일시스템 리사이즈

```bash
# 논리 볼륨 50G 증가
lvextend -L +50G /dev/Disks_DATA/DEV-WWW-data

# ext4 파일시스템 크기 동기화
resize2fs /dev/Disks_DATA/DEV-WWW-data
```

### 🧭 GPT 파티션 초기화 및 설정

```bash
# /dev/sdb에 GPT 파티션 테이블 생성 및 전체 영역 파티션 지정
parted --script /dev/sdb mklabel gpt mkpart primary 0 100% p

# /dev/sdc 및 /dev/sdd도 동일 방식으로 파티션 구성
parted -s /dev/sdc mklabel gpt mkpart primary 0 100% p
parted -s /dev/sdd mklabel gpt mkpart primary 0 100% p
```

### 🧼 필터 오류 시 디스크 초기화 → PV 생성 → VG 생성

```bash
vgcreate Disks /dev/sdb
  Device /dev/sdb excluded by a filter.
```
>
```bash
wipefs -a /dev/sdb
/dev/sdb: 8 bytes were erased at offset 0x00000200 (gpt): 45 46 49 20 50 41 52 54
/dev/sdb: 8 bytes were erased at offset 0xb54ba25e00 (gpt): 45 46 49 20 50 41 52 54
/dev/sdb: 2 bytes were erased at offset 0x000001fe (PMBR): 55 aa
/dev/sdb: calling ioctl to re-read partition table: Success
```
>
```bash
pvcreate /dev/sdb
```
>
```bash
vgcreate Disks /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```
> - 오류 발생시
{: .prompt-info }

### 🛠️ 장치 재스캔 (디스크 인식 오류 시)

```bash
# 디스크 혹은 파티션이 커널에 인식되지 않을 때 강제 재스캔

echo 1 > /sys/block/sda/device/rescan
# 또는 다른 장치명일 경우
echo 1 > /sys/block/xvda3/device/rescan
echo 1 > /sys/class/block/xvda3/device/rescan

# sudo가 필요할 경우
echo 1 | sudo tee /sys/block/xvda3/device/rescan
echo 1 | sudo tee /sys/class/block/xvda3/device/rescan
```

- wipefs -a: 디스크 메타데이터(GPT, LVM, RAID, ZFS 등) 완전 제거
- pvcreate: LVM 초기화, fdisk나 parted로 파티션 후 수행
- vgcreate: 새로운 LVM 그룹 생성
- lvcreate: 디스크 공간을 논리적으로 쪼개서 할당
- resize2fs: ext4 전용 파일시스템 확장

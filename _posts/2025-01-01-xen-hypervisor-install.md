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

### 저장소 생성
기본적으로 /dev/sda가 OS영역으로 할당되어 있음

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

3. 오류 발생시
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

### 패키지 설치
필수 패키지를 설치합니다.

```bash
sudo apt update
sudo apt install -y xen-hypervisor-4.17-amd64 xen-utils-4.17 xen-tools
```

### cfg 파일 설정
`/etc/default/grub.d/xen.cfg`{: .filepath} 파일을 Xen이 기본 GRUB 부트 항목을 오버라이드하도록 설정합니다.

```cfg
sed -i 's/#XEN_OVERRIDE_GRUB_DEFAULT=.*/XEN_OVERRIDE_GRUB_DEFAULT=1/' /etc/default/grub.d/xen.cfg
```
{: file='xen.cfg'}

Xen 하이퍼바이저 커널 옵션 추가

```cfg
sed -i -e 's/#GRUB_CMDLINE_XEN_DEFAULT=.*/GRUB_CMDLINE_XEN_DEFAULT="dom0_mem=16G,max:32G dom0_max_vcpus=1-4 dom0_vcpus_pin iommu=1 msi=1 pci_pt_e820_access=on watchdog ucode=scan crashkernel=256M,below=4G console=vga,com1 vga=mode-0x0314 com1=115200,8n1"/' \
       -e 's/#GRUB_CMDLINE_XEN=.*/GRUB_CMDLINE_XEN=""/' /etc/default/grub.d/xen.cfg
```
{: file='xen.cfg'}

### grub 업데이트

```bash
update-grub
```

### 시스 재부팅

```bash
reboot
```

### 주요 명령어
1. `xl list`: Xen 하이퍼바이저에서 실행 중인 도메인(가상 머신) 상태를 확인합니다.
2. `xl info`: Xen 하이퍼바이저의 전반적인 상태 및 설정 정보를 출력합니다.
3. `xl create`: Xen 가상 머신(도메인)을 생성하고 실행합니다.
4. `xl shutdown`: Xen 가상 머신(도메인)을 안전하게 종료합니다.
5. `xl destroy`: Xen 가상 머신(도메인)을 강제로 종료합니다.
6. `xl mem-set`: Xen 가상 머신(도메인)에 할당된 메모리를 동적으로 조정합니다.

```bash
xl list
```

```bash
xl info
```

```bash
xl create /etc/xen/noble.cfg
```

```bash
xl shutdown <domain>
```

```bash
xl destroy <domain>
```

```bash
xl mem-set <domain> <memory-size>
```

## 구성 파일 생성

### skel 설정
Xen-tools에서 VM을 생성할 때 사용할 기본 디렉토리 구조를 설정합니다.

```bash
mkdir -pv /etc/xen-tools/skel/root/.ssh
chown -R root:root /etc/xen-tools/skel/root/.ssh
cp -v /etc/skel/.bashrc /etc/xen-tools/skel/root
cp -v ~/.ssh/{authorized_keys,config} /etc/xen-tools/skel/root/.ssh
tree -apf /etc/xen-tools/skel/
```

### 패키지 설치 및 설정
Xen VM에 필요한 패키지를 설치하고, 다양한 환경 설정을 자동으로 처리하는 `/etc/xen-tools/role.d/custom_role`{: .filepath} 스크립트를 작성합니다.

```bash
#!/bin/bash
#
# Required packages installation script for Xen VM
#
# This script installs a list of essential packages and configures zsh plugins, SSH settings, timezone, locale, and user groups.

prefix=$1

# Check if the common functions file exists, which includes the installDebianPackage function.
if [ -e /usr/share/xen-tools/common.sh ]; then
    . /usr/share/xen-tools/common.sh
else
    echo "Installation problem: common.sh not found"
    exit 1
fi

chroot ${prefix} /bin/bash -c "set -o xtrace"

# List of required packages to be installed
packages=(
  alien
  apt-rdepends
  bash-completion
  colordiff
  command-not-found
  curl
  dnsutils
  dstat
  ethtool
  file
  gcc
  gdb
  git
  hardinfo
  hdparm
  htop
  ifenslave
  iftop
  imvirt
  iotop
  iptables
  iptraf-ng
  jq
  landscape-common
  lshw
  lsof
  lsscsi
  lvm2
  multitail
  mtr
  neofetch
  neovim
  net-tools
  network-manager
  nfs-common
  nscd
  ntp
  openvswitch-switch
  parted
  psmisc
  pigz
  pv
  python3
  python3-dev
  python3-pip
  python3-pipdeptree
  rename
  rsync
  screen
  smartmontools
  software-properties-common
  speedtest-cli
  spfquery
  ssh-askpass
  sysstat
  tcpdump
  tcpflow
  telnet
  tmux
  traceroute
  tree
  ufw
  vim
  vim-gtk
  virt-what
  vnstat
  wget
  whois
  xonsh
  xxhash
)

# Update APT lists
chroot ${prefix} /usr/bin/apt-get update

# Install each package from the list
for package in "${packages[@]}"; do
  installDebianPackage ${prefix} ${package}
done

# SSH configuration adjustments
chroot ${prefix} /bin/bash -c "sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config"
chroot ${prefix} /bin/bash -c "sed -i '/#PubkeyAuthentication yes/a PubkeyAcceptedAlgorithms +ssh-rsa' /etc/ssh/sshd_config"

# Timezone configuration
chroot ${prefix} /bin/bash -c "rm -f /etc/localtime && ln -s /usr/share/zoneinfo/Asia/Seoul /etc/localtime && echo 'Asia/Seoul' > /etc/timezone"

# Locale configuration
chroot ${prefix} /bin/bash -c "/usr/bin/localectl set-locale LANG=en_US.UTF-8"
chroot ${prefix} /bin/bash -c "sed -i 's/# ko_KR.UTF-8 UTF-8/ko_KR.UTF-8 UTF-8/' /etc/locale.gen && /usr/sbin/locale-gen"

# Change group and user IDs for nobody
chroot ${prefix} /bin/bash -c "groupadd -fg 99 nobody"
chroot ${prefix} /bin/bash -c "sed -i -e 's/^\(nobody:[^:]\):[0-9]*:[0-9]*:/\1:99:99:/' /etc/passwd"

# systemctl enable --now systemd-networkd-wait-online.service
# Disable and stop systemd-networkd-wait-online.service
chroot ${prefix} /bin/bash -c "rm -v /etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service"
# systemctl mask systemd-networkd-wait-online.service
chroot ${prefix} /bin/bash -c "ln -sfv /dev/null /etc/systemd/system/systemd-networkd-wait-online.service"

# ZSH plugins and configurations
chroot ${prefix} /bin/bash -c "git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf && ~/.fzf/install --all"

# Sysstat configuration (after package installation)
chroot ${prefix} /bin/bash -c "sed -i 's/ENABLED=\"[^\"]*\"/ENABLED=\"true\"/' /etc/default/sysstat"
chroot ${prefix} /bin/bash -c "sed -i -e 's/^HISTORY=.*/HISTORY=31/' -e 's/^COMPRESSAFTER=.*/COMPRESSAFTER=31/' /etc/sysstat/sysstat"
chroot ${prefix} /bin/bash -c "cat << EOF | sudo tee /etc/cron.d/sysstat
# The first element of the path is a directory where the debian-sa1
# script is located
PATH=/usr/lib/sysstat:/usr/sbin:/usr/sbin:/usr/bin:/sbin:/bin

# Activity reports every 5 minutes everyday
*/5 * * * * root command -v debian-sa1 > /dev/null && debian-sa1 1 1

# Additional run at 23:59 to rotate the statistics file
59 23 * * * root command -v debian-sa1 > /dev/null && debian-sa1 60 2
EOF"
chroot ${prefix} /bin/bash -c "/usr/sbin/dpkg-reconfigure -f noninteractive sysstat"


# NTP configuration changes (after package installation)
chroot ${prefix} /bin/bash -c "sed -i 's/pool 0.ubuntu.pool.ntp.org iburst/server time1.google.com iburst/' /etc/ntp.conf"
chroot ${prefix} /bin/bash -c "sed -i 's/pool 1.ubuntu.pool.ntp.org iburst/server time2.google.com iburst/' /etc/ntp.conf"
chroot ${prefix} /bin/bash -c "sed -i 's/pool 2.ubuntu.pool.ntp.org iburst/server time3.google.com iburst/' /etc/ntp.conf"
chroot ${prefix} /bin/bash -c "sed -i 's/pool 3.ubuntu.pool.ntp.org iburst/server time4.google.com iburst/' /etc/ntp.conf"

chroot ${prefix} /bin/bash -c "set +o xtrace"
```

## VM 생성

### Domain-U 생성
1. LVM 방식
  - cfg 파일
2. IMG 방식
  - cfg 파일

```bash
xen-create-image    --hostname=noble \
                    --dist=noble \
                    --memory=16G \
                    --maxmem=64G \
                    --vcpus=44 \
                    --bridge=xenbr0 \
                    --dhcp \
                    --size=50G \
                    --swap=1G \
                    --lvm=Disks \
                    --force \
                    --pygrub \
                    --role='custom_role' \
                    --password=1234 \
                    --extension=
```

```bash
#
# Configuration file for the Xen instance ubuntu-img, created
# by xen-tools 4.9.2 on Tue Feb 13 09:38:46 2024.
#

#
#  Kernel + memory size
#


bootloader = 'pygrub'

cpus        = [ '4-47' ]
vcpus       = '44'
memory      = '16384'
maxmem      = '65536'

#
#  Disk device(s).
#
root        = '/dev/xvda2 ro'
disk        = [
                  'phy:/dev/Disks/noble-disk,xvda2,w',
                  'phy:/dev/Disks/noble-swap,xvda1,w',
              ]


#
#  Physical volumes
#


#
#  Hostname
#
name        = 'noble'

#
#  Networking
#
dhcp        = 'dhcp'
vif         = [ 'bridge=xenbr0' ]

#
#  Behaviour
#
on_poweroff = 'destroy'
on_reboot   = 'restart'
on_crash    = 'restart'
```

```bash
xen-create-image    --hostname=noble \
                    --dist=noble \
                    --memory=16G \
                    --maxmem=64G \
                    --vcpus=44 \
                    --bridge=xenbr0 \
                    --dhcp \
                    --size=50G \
                    --swap=1G \
                    --dir=/data/volumes \
                    --force \
                    --pygrub \
                    --role='custom_role' \
                    --password=1234 \
                    --extension=
```

```bash
#
# Configuration file for the Xen instance ubuntu-img, created
# by xen-tools 4.9.2 on Tue Feb 13 09:38:46 2024.
#

#
#  Kernel + memory size
#


bootloader = 'pygrub'

cpu         = '4-47'
vcpus       = '44'
memory      = '16384'
maxmem      = '65536'

#
#  Disk device(s).
#
root        = '/dev/xvda2 ro'
disk        = [
                  'file:/data/volumes/domains/jammy/disk.img,xvda2,w',
                  'file:/data/volumes/domains/jammy/swap.img,xvda1,w',
              ]


#
#  Physical volumes
#


#
#  Hostname
#
name        = 'noble'

#
#  Networking
#
dhcp        = 'dhcp'
vif         = [ 'bridge=xenbr0' ]

#
#  Behaviour
#
on_poweroff = 'destroy'
on_reboot   = 'restart'
on_crash    = 'restart'
```

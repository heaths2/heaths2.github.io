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

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

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

</details>

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
# 또는
systemctl restart nfs-kernel-server
```

## iPXE 설치

### iPXE 설치 및 환경 구성
1. 필수 패키지 설치
2. iPXE 소스 코드 다운로드
3. iPXE 설정 변경
- DOWNLOAD_PROTO_NFS: NFS(Network File System) 다운로드 지원.
- PING_CMD: 네트워크 연결 상태를 확인하기 위한 ping 명령 활성화.
- IPSTAT_CMD: 네트워크 통계 출력 명령 활성화.
- REBOOT_CMD: 재부팅 명령 활성화.
- POWEROFF: 전원 종료 명령 활성화.
- CONSOLE_CMD: 콘솔 명령어 활성화.
- 프레임 버퍼 콘솔 지원을 활성화.
4. 설정 확인
5. iPXE 바이너리 빌드
6. TFTP 디렉토리 구성
7. iPXE 초기 스크립트 작성
8. iPXE 부트 메뉴 작성

```bash
sudo apt install build-essential liblzma-dev isolinux git
```

```bash
git clone https://github.com/ipxe/ipxe.git
cd ipxe/src
```

```bash
sed -i.bak -e 's/#undef\tDOWNLOAD_PROTO_NFS/#define\tDOWNLOAD_PROTO_NFS/' \
       -e 's/\/\/#define\ PING_CMD/#define\ PING_CMD/' \
       -e 's/\/\/#define\ IPSTAT_CMD/#define\ IPSTAT_CMD/' \
       -e 's/\/\/#define\ REBOOT_CMD/#define\ REBOOT_CMD/' \
       -e 's/\/\/#define\ POWEROFF/#define\ POWEROFF/' \
       -e 's/\/\/#define\ CONSOLE_CMD/#define\ CONSOLE_CMD/' config/general.h
```

```bash
sed -i 's/\/\/#define[[:space:]]\+CONSOLE_FRAMEBUFFER/#define       CONSOLE_FRAMEBUFFER/' config/console.h
```

```bash
grep -E 'DOWNLOAD_PROTO_NFS|PING_CMD|IPSTAT_CMD|REBOOT_CMD|POWEROFF_CMD|CONSOLE_CMD/' config/general.h
grep 'CONSOLE_FRAMEBUFFER' config/console.h
```

```bash
make bin/ipxe.pxe bin/undionly.kpxe bin/undionly.kkpxe bin/undionly.kkkpxe bin-x86_64-efi/ipxe.efi EMBED=/root/ipxe/src/embed.ipxe
```

```bash
mkdir -pv /srv/tftp/ipxe/{bin,src}/
cp -av bin/{ipxe.pxe,undionly.kpxe,undionly.kkpxe,undionly.kkkpxe} bin-x86_64-efi/ipxe.efi /srv/tftp/ipxe/bin/
wget http://boot.ipxe.org/ipxe.png -O /srv/tftp/ipxe/src/ipxe.png
```

```bash
tee /root/ipxe/src/embed.ipxe << EOF
#!ipxe
# DHCP
dhcp
chain tftp://\${next-server}/main.ipxe
EOF
```

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```bash
#!ipxe
# Background Images
console --x 1024 --y 768 --picture ipxe/src/ipxe.png --left 32 --right 32 --top 32 --bottom 48

# Set Color Fonts
set esc:hex 1b
set bold ${esc:string}[1m
set boldoff ${esc:string}[22m
set fg_off ${esc:string}[0m
set fg_red ${esc:string}[31m
set fg_gre ${esc:string}[32m
set fg_cya ${esc:string}[36m
set fg_whi ${esc:string}[37m

# Set NFS strings
set nfs-server          ${next-server}
set nfs-mount           /srv/tftp
set nfs-path            nfs://${nfs-server}${nfs-mount}
set nfs-root            ${nfs-server}:${nfs-mount}

# HTTP and iSCSI
set iscsi-server        ${next-server}
set http-root           http://${next-server}:3259

:start
menu iPXE boot menu options
item --gap --                   ------------------------- Local boot options ------------------------------
item            localboot               Boot to local drive
item --gap --                   ------------------------- Network boot options ----------------------------
item            ubuntu24.04             Install Ubuntu 24.04
item            ubuntu22.04             Install Ubuntu 22.04
item            ubuntu20.04             Install Ubuntu 20.04
item            windows10               Install Windows10
item            restoredisk-u24         Install Clonezilla Restoredisk u24-BareMetal
item            restoredisk-xen24       Install Clonezilla Restoredisk u24-XEN
item            restoredisk-u22         Install Clonezilla Restoredisk u22-BareMetal
item            restoredisk-xen22       Install Clonezilla Restoredisk u22-XEN
item            restoredisk-u20         Install Clonezilla Restoredisk u20-BareMetal
item            restoredisk-xen20       Install Clonezilla Restoredisk u20-XEN
item --gap --                   ------------------------- HPE options -------------------------------------
item            ssa2.60                 Go to HPE Smart Storage Administrator 2.60
item            ssa4.15                 Go to HPE Smart Storage Administrator 4.15
item            gen8                    Go to HPE Service Pack for ProLiant Gen8
item            gen9                    Go to HPE Service Pack for ProLiant Gen9
item --gap --                   ------------------------- Advanced options --------------------------------
item --key c    clonezilla              Go to Clonezilla Live
item --key s    shell                   Go to iPXE Shell
item --key r    reboot                  Reboot
item
item --key x    exit                    Exit iPXE and continue BIOS boot
choose --default localboot --timeout 10000 target && goto ${target}

:localboot
echo ${fg_gre}Continue${fg_off} booting to local drive
goto exit

:shell
echo Type "exit" to return to menu
shell
goto start

:reboot
reboot

:exit
exit

###
### Custom menu entries
###

:clonezilla
kernel clonezilla/live/vmlinuz
initrd clonezilla/live/initrd.img
imgargs vmlinuz boot=live username=user union=overlay components noswap noprompt keyboard-layouts=us locales=en_US.UTF-8 fetch=tftp://${nfs-server}/clonezilla/live/filesystem.squashfs video=1024x768
boot

:restoredisk-u24
kernel clonezilla/live/vmlinuz
initrd clonezilla/live/initrd.img
imgargs vmlinuz boot=live username=user union=overlay config components quiet noswap edd=on nomodeset enforcing=0 noeject fetch=tftp://${nfs-server}/clonezilla/live/filesystem.squashfs ocs_prerun="dhclient -v" ocs_prerun1="echo '1234' | sshfs root@${nfs-server}:/home/partimag /home/partimag -p 22 -o noatime -o ssh_command='ssh -oStrictHostKeyChecking=No' -o password_stdin" ocs_live_run="/usr/sbin/ocs-sr -g auto -e1 auto -e2 -r -j2 -icds -k1 -p reboot restoredisk u24-BareMetal sda" keyboard-layouts=NONE ocs_live_batch="no" locales="en_US.UTF-8" nolocales video=1024x768
boot

:restoredisk-xen24
kernel clonezilla/live/vmlinuz
initrd clonezilla/live/initrd.img
imgargs vmlinuz boot=live username=user union=overlay config components quiet noswap edd=on nomodeset enforcing=0 noeject fetch=tftp://${nfs-server}/clonezilla/live/filesystem.squashfs ocs_prerun="dhclient -v" ocs_prerun1="echo '1234' | sshfs root@${nfs-server}:/home/partimag /home/partimag -p 22 -o noatime -o ssh_command='ssh -oStrictHostKeyChecking=No' -o password_stdin" ocs_live_run="/usr/sbin/ocs-sr -g auto -e1 auto -e2 -r -j2 -icds -k1 -p reboot restoredisk u24-XEN sda" keyboard-layouts=NONE ocs_live_batch="no" locales="en_US.UTF-8" nolocales video=1024x768
boot

:restoredisk-u22
kernel clonezilla/live/vmlinuz
initrd clonezilla/live/initrd.img
imgargs vmlinuz boot=live username=user union=overlay config components quiet noswap edd=on nomodeset enforcing=0 noeject fetch=tftp://${nfs-server}/clonezilla/live/filesystem.squashfs ocs_prerun="dhclient -v" ocs_prerun1="echo '1234' | sshfs root@${nfs-server}:/home/partimag /home/partimag -p 22 -o noatime -o ssh_command='ssh -oStrictHostKeyChecking=No' -o password_stdin" ocs_live_run="/usr/sbin/ocs-sr -g auto -e1 auto -e2 -r -j2 -icds -k1 -p reboot restoredisk u22-BareMetal sda" keyboard-layouts=NONE ocs_live_batch="no" locales="en_US.UTF-8" nolocales video=1024x768
boot

:restoredisk-xen22
kernel clonezilla/live/vmlinuz
initrd clonezilla/live/initrd.img
imgargs vmlinuz boot=live username=user union=overlay config components quiet noswap edd=on nomodeset enforcing=0 noeject fetch=tftp://${nfs-server}/clonezilla/live/filesystem.squashfs ocs_prerun="dhclient -v" ocs_prerun1="echo '1234' | sshfs root@${nfs-server}:/home/partimag /home/partimag -p 22 -o noatime -o ssh_command='ssh -oStrictHostKeyChecking=No' -o password_stdin" ocs_live_run="/usr/sbin/ocs-sr -g auto -e1 auto -e2 -r -j2 -icds -k1 -p reboot restoredisk u22-XEN sda" keyboard-layouts=NONE ocs_live_batch="no" locales="en_US.UTF-8" nolocales video=1024x768
boot

:restoredisk-u20
kernel clonezilla/live/vmlinuz
initrd clonezilla/live/initrd.img
imgargs vmlinuz boot=live username=user union=overlay config components quiet noswap edd=on nomodeset enforcing=0 noeject fetch=tftp://${nfs-server}/clonezilla/live/filesystem.squashfs ocs_prerun="dhclient -v" ocs_prerun1="echo '1234' | sshfs root@${nfs-server}:/home/partimag /home/partimag -p 22 -o noatime -o ssh_command='ssh -oStrictHostKeyChecking=No' -o password_stdin" ocs_live_run="/usr/sbin/ocs-sr -g auto -e1 auto -e2 -r -j2 -icds -k1 -p reboot restoredisk u20-BareMetal sda" keyboard-layouts=NONE ocs_live_batch="no" locales="en_US.UTF-8" nolocales video=1024x768
boot

:restoredisk-xen20
kernel clonezilla/live/vmlinuz
initrd clonezilla/live/initrd.img
imgargs vmlinuz boot=live username=user union=overlay config components quiet noswap edd=on nomodeset enforcing=0 noeject fetch=tftp://${nfs-server}/clonezilla/live/filesystem.squashfs ocs_prerun="dhclient -v" ocs_prerun1="echo '1234' | sshfs root@${nfs-server}:/home/partimag /home/partimag -p 22 -o noatime -o ssh_command='ssh -oStrictHostKeyChecking=No' -o password_stdin" ocs_live_run="/usr/sbin/ocs-sr -g auto -e1 auto -e2 -r -j2 -icds -k1 -p reboot restoredisk u20-XEN sda" keyboard-layouts=NONE ocs_live_batch="no" locales="en_US.UTF-8" nolocales video=1024x768
boot

:ubuntu22.04
kernel ubuntu22.04/casper/vmlinuz
initrd ubuntu22.04/casper/initrd
imgargs vmlinuz initrd=initrd ip=dhcp nfsroot=${nfs-root}/ubuntu24.04 netboot=nfs boot=casper maybe-ubiquity quiet splash video=1024x768 --- 
boot

:ubuntu22.04
kernel ubuntu22.04/casper/vmlinuz
initrd ubuntu22.04/casper/initrd
imgargs vmlinuz initrd=initrd ip=dhcp nfsroot=${nfs-root}/ubuntu22.04 netboot=nfs boot=casper maybe-ubiquity quiet splash video=1024x768 --- 
boot

:ubuntu20.04
kernel ubuntu20.04/casper/vmlinuz
initrd ubuntu20.04/casper/initrd
imgargs vmlinuz initrd=initrd ip=dhcp nfsroot=${nfs-root}/ubuntu20.04 netboot=nfs boot=casper maybe-ubiquity quiet splash video=1024x768
boot

:ssa2.60
kernel ssa2.60/system/vmlinuz
initrd ssa2.60/system/initrd.img
imgargs vmlinuz initrd=initrd.img media=cdrom rw root=/dev/ram0 ramdisk_size=257144 init=/bin/init loglevel=3 ide=nodma ide=noraid pnpbios=off splash=silent showopts TYPE=MANUAL iso1mnt=/mnt/ssa2.60 iso1=nfs://${nfs-server}/srv/tftp/ssaoffline-2.60-18.0.iso iso1opts=timeo=120,nolock,bg,ro video=1024x768
boot

:ssa4.15
kernel ssa4.15/system/vmlinuz
initrd ssa4.15/system/initrd.img
imgargs vmlinuz initrd=initrd.img media=cdrom rw root=/dev/ram0 ramdisk_size=257144 init=/bin/init loglevel=3 ide=nodma ide=noraid pnpbios=off splash=silent showopts TYPE=MANUAL iso1mnt=/mnt/ssa4.15 iso1=nfs://${nfs-server}/srv/tftp/ssaoffline-4.15-6.0.iso iso1opts=timeo=120,nolock,bg,ro video=1024x768
boot

:gen8
kernel spp8/system/vmlinuz
initrd spp8/system/initrd.img
imgargs vmlinuz initrd=initrd.img media=net root=/dev/ram0 splash quiet hp_fibre showopts TYPE=AUTOMATIC AUTOPOWEROFFONSUCCESS=no AUTOREBOOTONSUCCESS=yes iso1=nfs://${nfs-server}/srv/tftp/spp8/spp8.iso iso1mnt=/mnt/bootdevice video=1024x768
boot

:gen9
kernel spp9/system/vmlinuz
initrd spp9/system/initrd.img
imgargs vmlinuz initrd=initrd.img media=net root=/dev/ram0 splash quiet hp_fibre showopts TYPE=AUTOMATIC AUTOPOWEROFFONSUCCESS=no AUTOREBOOTONSUCCESS=yes iso1=nfs://${nfs-server}/srv/tftp/spp9/spp9.iso iso1mnt=/mnt/bootdevice video=1024x768
boot
```

</details>

### 테스트 및 검증

## 참조
- [iPXE 공식 사이트](https://ipxe.org/)
- [FOG Project Wiki 문서](https://wiki.fogproject.org/wiki/index.php/BIOS_and_UEFI_Co-Existence)
- [GitHub Gist 사용자 정의 메뉴 참조 문서](https://gist.github.com/rikka0w0/50895b82cbec8a3a1e8c7707479824c1)


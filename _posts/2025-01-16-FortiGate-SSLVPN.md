---
title: FortiGate SSL VPN 설정
author: G.G
date: 2025-01-16 11:39:00 +0900
categories: [Blog, Provisioning]
tags: [Fortinet, Fortigate, VPN]
---

## 📘 개요
FortiGate SSLVPN은 Fortinet 방화벽을 통해 안전한 원격 접속을 제공하는 VPN 솔루션입니다. SSL 기반 암호화를 활용하여 인터넷을 통해 사내 네트워크에 원격으로 안전하게 연결할 수 있습니다.
이 매뉴얼은 FortiGate에서 SSLVPN을 설정하고 관리하는 방법을 단계별로 안내합니다.

## 목적
1. FortiGate SSLVPN 설정 매뉴얼은 다음과 같은 목적을 갖습니다:
- FortiGate 방화벽을 이용해 원격 근무자들이 사내 네트워크에 안전하게 접속할 수 있도록 지원.
- 네트워크 보안을 유지하면서도 효율적인 원격 접속을 구현.
- 관리자가 FortiGate SSLVPN을 구성하고, 문제를 해결하며, 최적화할 수 있도록 도움 제공.
## 특징
FortiGate SSLVPN은 다음과 같은 주요 특징을 제공합니다:

1. 보안
- SSL/TLS 기반 암호화를 사용하여 전송 중 데이터 보호.
- 두 단계 인증(2FA)을 통한 보안 강화.
- IP 및 사용자 기반 접근 제어 정책 제공.
2. 유연성
- 웹 브라우저 기반 포털 또는 FortiClient 애플리케이션을 사용한 접속.
- 특정 애플리케이션 트래픽만 VPN을 통해 전달하는 스플릿 터널링(Split Tunneling) 지원.
3. 확장성
- 다양한 디바이스 및 플랫폼(Windows, macOS, Linux, Android, iOS) 지원.
- 원격 접속 사용자 수에 따라 유연한 확장 가능.
4. 관리 편의성
- 중앙집중식 관리 인터페이스(FortiGate GUI)를 통한 설정 및 모니터링.
- 사용자 세션 모니터링 및 실시간 로그 분석.
- 통합된 Fortinet 보안 솔루션과의 연동.

## SSLVPN 설정

### SSLVPN 사용자 IP 풀 설정
- SSLVPN 사용자가 할당받을 IP 주소 범위를 지정합니다.
- 여기서는 192.168.200.0/24 대역을 사용합니다.

```bash
config firewall address
    edit "SSLVPN address"
        set subnet 192.168.200.0 255.255.255.0
    next
end
```

### SSLVPN 포털 설정
- SSLVPN 사용자를 위한 포털 설정입니다.
- 터널 모드를 활성화하고, MAC 주소 인증을 설정합니다.

```bash
config vpn ssl web portal
    edit "Infra-SSLVPN"
        set tunnel-mode enable
        set ip-pools "SSLVPN address"
        set mac-addr-check enable
        config mac-addr-check-rule
            edit "Allowed-VPN-MAC"
                set mac-addr-list aa:bb:cc:dd:ee:ff
            next
        set mac-addr-action allow
        end
    next
end
```

### 방화벽 정책 설정
- SSLVPN에서 내부 네트워크로의 접근을 허용하는 정책입니다.

```bash
config firewall policy
    edit 4
        set srcintf "ssl.root"
        set dstintf "Internal"
        set action accept
        set srcaddr "SSLVPN address"
        set dstaddr "VLAN-500 address"
        set schedule "always"
        set service "ALL"
        set logtraffic all
        set groups "Radius Authentication"
    next
end
```

### SSLVPN 포털 상세 설정 확인
- "Infra-SSLVPN" 포털의 전체 설정을 확인합니다.

```bash
Infra # show full-configuration vpn ssl web portal Infra-SSLVPN
config vpn ssl web portal
    edit "Infra-SSLVPN"
        set tunnel-mode enable
        set ipv6-tunnel-mode disable
        set web-mode disable
        set allow-user-access web ftp smb sftp telnet ssh vnc rdp ping
        set limit-user-logins disable
        set forticlient-download enable
        set ip-mode range
        set auto-connect disable
        set keep-alive disable
        set save-password disable
        set ip-pools "SSLVPN address"
        set split-tunneling enable
        set split-tunneling-routing-negate disable
        set dns-server1 0.0.0.0
        set dns-server2 0.0.0.0
        set dns-suffix ''
        set wins-server1 0.0.0.0
        set wins-server2 0.0.0.0
        set dhcp-ra-giaddr 0.0.0.0
        set client-src-range disable
        set host-check none
        set mac-addr-check enable
        set mac-addr-action allow
        config mac-addr-check-rule
            edit "Allowed-VPN-MAC"
                set mac-addr-mask 48
                set mac-addr-list aa:bb:cc:dd:ee:ff
            next
        end
        set os-check disable
        set forticlient-download-method direct
        set customize-forticlient-download-url disable
        set skip-check-for-unsupported-os enable
        set skip-check-for-browser enable
    next
end

Infra #
```

> - `show full-configuration vpn ssl web portal Infra-SSLVPN` 명령어
{: .prompt-tip }

### SSLVPN 서버 설정
- SSL 인증서 및 사용자, 인증 그룹추가

```bash
config vpn ssl settings
    set servercert "Fortinet_Factory"
    set tunnel-ip-pools "SSLVPN address"
    set dns-server1 168.126.63.1
    set dns-server2 168.126.63.2
    set port 443
    set source-interface "External"
    set source-address "all"
    set source-address6 "all"
    set default-portal "tunnel-access"
    config authentication-rule
        edit 1
            set groups "Radius Authentication" "SSL-VPN"
            set portal "Infra-SSLVPN"
        next
    end
end
```

### 디버깅 명령어
- SSLVPN 디버깅을 위한 명령어입니다.
- 디버그 로그를 활성화하여 문제를 진단할 수 있습니다.

```bash
diagnose debug application sslvpn -1
diagnose debug console timestamp enable
diagnose debug enable
```

## 참조
> - [FortiGate 공식문서](https://docs.fortinet.com/document/fortigate/7.4.4/administration-guide/032970/configuring-os-and-host-check)

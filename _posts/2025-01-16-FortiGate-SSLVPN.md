---
title: FortiGate SSL VPN 설정
author: G.G
date: 2025-01-16 11:39:00 +0900
categories: [Blog, Provisioning]
tags: [fortinet, fortigate, vpn]
---

## 개요

> - FortiGate SSL VPN 설정
> - [FortiGate RADIUS 연동](https://heaths2.github.io/docs/category/blog/b0147)
> - [FortiGate 공식문서](https://docs.fortinet.com/document/fortigate/7.4.4/administration-guide/032970/configuring-os-and-host-check)
{: .new }

### SSL VPN 설정

- SSL VPN Address 객체 생성

```bash
config firewall address
    edit "SSLVPN address"
        set subnet 192.168.200.0 255.255.255.0
    next
end
```

- SSL VPN Tunnel Mode 설정

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

- SSL VPN 방화벽 정책 설정

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

- SSL VPN 설정 확인

<details markdown="block">
  <summary>
    코드
  </summary>
  {: .text-delta .label .label-green }

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

</details>

- Client SSL VPN 접속
  
![image](https://github.com/heaths2/heaths2.github.io/assets/36792594/4f0fb282-3bd8-4920-984e-8b0eedb88fe7)

### SSL VPN 디버깅

- SSL VPN 디버깅

```bash
diagnose debug application sslvpn -1
diagnose debug console timestamp enable
diagnose debug enable
```

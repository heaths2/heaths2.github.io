---
title: Cisco Switch Bandwidth Limit 설정
author: G.G
date: 2025-01-08 10:36:00 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Cisco, QoS, Bandwidth]
---

## 📘 개요
Cisco 스위치에서 Bandwidth Limit 설정은 네트워크 트래픽을 효율적으로 관리하기 위한 QoS(Quality of Service) 기능의 일부입니다. 특정 시간대(예: 00:00 ~ 18:00) 동안 대역폭을 제한하여 네트워크 리소스의 불균형 사용을 방지하거나, 우선순위가 낮은 트래픽을 제어하는 데 사용됩니다. 이러한 설정은 네트워크 성능 최적화 및 안정성을 높이는 데 기여합니다.

## 특징
1. 시간 기반 대역폭 제한
- 시간대 스케줄링 기능을 사용하여 특정 시간 범위 동안만 대역폭 제한을 적용.
- 예: 업무 시간(00:00 ~ 18:00) 동안 제한을 적용하고, 비업무 시간에는 제한 해제.

2. 정책 기반 대역폭 제어
- ACL(Access Control List) 및 QoS 정책을 사용해 특정 트래픽이나 포트에 대역폭 제한을 설정.
- 네트워크 서비스의 우선순위를 설정하거나 제한된 리소스를 효과적으로 배분 가능.

3. 효율적인 네트워크 리소스 관리
- 과도한 대역폭 사용을 방지하고, 네트워크 혼잡 상황을 완화.
- 중요 트래픽(예: VoIP, 화상회의 등) 보호 가능.

4. 다양한 적용 대상
- 특정 VLAN, 인터페이스, 또는 IP 주소를 대상으로 대역폭 제한 적용 가능.
- 네트워크의 유연한 정책 구현을 지원.

5. 제어 가능한 트래픽 속도
- Mbps 또는 Kbps 단위로 대역폭 제한을 세부적으로 설정 가능.

## QoS Bandwidth Limit 설정

### QoS 설정
1. QoS 설정
2. QoS 정책 설정
3. QoS 폴리싱 설정
4. ACL 설정
5. EEM 애플릿 설정

```bash
Switch# configure terminal
Switch(config)# mls qos
Switch(config)# class-map match-all QOS_CLASS_64KBPS
Switch(config-cmap)# match access-group name QOS_ACL_64KBPS
Switch(config-cmap)# exit
Switch(config)# policy-map QOS_POLICY_64KBPS
Switch(config-pmap)# class QOS_CLASS_64KBPS
Switch(config-pmap-c)# police 64000 64000 exceed-action drop
Switch(config-pmap-c)# exit
Switch(config-pmap)# exit
Switch(config)# ip access-list extended QOS_ACL_64KBPS
Switch(config-ext-nacl)# permit ip any any
Switch(config-ext-nacl)# exit
Switch(config)# event manager applet APPLY_QOS_POLICY_64KBPS
Switch(config-applet)# event timer cron cron-entry "02 16 * * *"
Switch(config-applet)# action 1.0 cli command "enable"
Switch(config-applet)# action 2.0 cli command "configure terminal"
Switch(config-applet)# action 3.0 cli command "interface GigabitEthernet2/0/3"
Switch(config-applet)# action 4.0 cli command "service-policy input QOS_POLICY_64KBPS"
Switch(config-applet)# event manager applet REMOVE_QOS_POLICY_64KBPS
Switch(config-applet)# event timer cron cron-entry "03 16 * * *"
Switch(config-applet)# action 1.0 cli command "enable"
Switch(config-applet)# action 2.0 cli command "configure terminal"
Switch(config-applet)# action 3.0 cli command "interface GigabitEthernet2/0/3"
Switch(config-applet)# action 4.0 cli command "no service-policy input QOS_POLICY_64KBPS"
Switch(config-applet)# end
Switch# write memory
```

### QoS 대역폭 제한 스크립트

```bash
configure terminal
mls qos
!
class-map match-all QOS_CLASS_64KBPS
 match access-group name QOS_ACL_64KBPS
!
policy-map QOS_POLICY_64KBPS
 class QOS_CLASS_64KBPS
  police 64000 64000 exceed-action drop
!
ip access-list extended QOS_ACL_64KBPS
 permit ip any any
!
event manager applet APPLY_QOS_POLICY_64KBPS
 event timer cron cron-entry "00 00 * * *"
 action 1.0 cli command "enable"
 action 2.0 cli command "configure terminal"
 action 3.0 cli command "interface GigabitEthernet2/0/3"
 action 4.0 cli command "service-policy input QOS_POLICY_64KBPS"
event manager applet REMOVE_QOS_POLICY_64KBPS
 event timer cron cron-entry "00 18 * * *"
 action 1.0 cli command "enable"
 action 2.0 cli command "configure terminal"
 action 3.0 cli command "interface GigabitEthernet2/0/3"
 action 4.0 cli command "no service-policy input QOS_POLICY_64KBPS"
!
```

### 설정 해제 스크립트 
1. ACL 설정
2. QoS 정책 설정
3. QoS 폴리싱 설정
4. EEM 애플릿 설정

```bash
configure terminal
no mls qos
no class-map match-all QOS_CLASS_64KBPS
no policy-map QOS_POLICY_64KBPS
no ip access-list extended QOS_ACL_64KBPS
no event manager applet APPLY_QOS_POLICY_64KBPS
no event manager applet REMOVE_QOS_POLICY_64KBPS
```

### QoS 정책 설정 확인

```bash
Switch# show class-map QOS_CLASS_64KBPS
```

<details markdown="block">
  <summary>
    코드
  </summary>
  {: .text-delta .label .label-green }

```bash
 Class Map match-all QOS_CLASS_64KBPS (id 1)
   Match access-group name QOS_ACL_64KBPS
```

</details>

### QoS 폴리싱 설정 확인

```bash
Switch# show policy-map QOS_POLICY_64KBPS
```

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>
  
```bash
  Policy Map QOS_POLICY_64KBPS
    Class QOS_CLASS_64KBPS
      police 64000 64000 exceed-action drop
```

</details>

### ACL 설정 확인

```bash
Switch# show access-lists QOS_ACL_64KBPS
```

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```bash
Extended IP access list QOS_ACL_64KBPS
    10 permit ip any any
```

</details>

```bash
Switch# show event manager policy registered
```

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```bash
Switch# show event manager policy registered                                                     
No.  Class     Type    Event Type          Trap  Time Registered           Name
1    applet    user    timer cron          Off   Fri Jun 14 16:50:53 2024  APPLY_QOS_POLICY_64KBPS
 cron entry {00 00 * * *}
 maxrun 20.000
 action 1.0 cli command "enable"
 action 2.0 cli command "configure terminal"
 action 3.0 cli command "interface GigabitEthernet2/0/3"
 action 4.0 cli command "service-policy input QOS_POLICY_64KBPS"

2    applet    user    timer cron          Off   Fri Jun 14 16:51:01 2024  REMOVE_QOS_POLICY_64KBPS
 cron entry {00 18 * * *}
 maxrun 20.000
 action 1.0 cli command "enable"
 action 2.0 cli command "configure terminal"
 action 3.0 cli command "interface GigabitEthernet2/0/3"
 action 4.0 cli command "no service-policy input QOS_POLICY_64KBPS"
```

</details>

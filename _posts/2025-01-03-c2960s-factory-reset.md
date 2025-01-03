---
title: Cisco Catalyst 2960 스위치 공장 초기화
author: G.G
date: 2025-01-03 20:50:00 +0900
categories: [Blog, Switch]
tags: [cisco, switch, c2960, catalyst 2960, L2]
---

## 개요
Cisco Catalyst 2960 시리즈 스위치는 네트워크 관리 및 보안을 지원하는 엔터프라이즈급 스위치로, 사용 중에 설정 오류나 새로운 환경으로의 전환이 필요한 경우 공장 초기화를 통해 기본 상태로 복원할 수 있습니다. 공장 초기화는 스위치의 현재 설정과 저장된 데이터를 삭제하고, 초기 상태로 복구하여 새롭게 설정할 수 있도록 합니다.

## 공장 초기화의 목적
1. 네트워크 재구성: 기존 설정을 제거하고 새로운 환경에 적합한 설정을 적용하기 위해.
2. 보안 강화: 민감한 설정 정보나 데이터가 유출되지 않도록 초기화.
3. 문제 해결: 설정 오류나 충돌로 인한 문제를 해결하기 위해 기본 상태로 복구.
4. 재판매 또는 재사용: 다른 사용자나 환경에서 장치를 사용할 수 있도록 초기 상태로 전환.
> - Cisco Switch Factory Reset을 숙지한다.
> - [공장 초기화](https://niksec.com/how-to-reset-cisco-catalyst-2960-switches-to-factory-default) 참고 사이트
> - [C2960](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst2960/hardware/installation/guide_stack/2960SHIG/HIGOVERV.html) 참고 사이트
{: .new }

공장 초기화
> - `mode` button를 누른 상태로 전원을 킨다.
{: .important }

![image](https://user-images.githubusercontent.com/36792594/208580157-9591f2de-ded0-49ba-8af0-6464bfd93294.png)

![IMG-001](https://user-images.githubusercontent.com/36792594/208579393-78cd1c27-0f8f-4007-8894-7f8fe9977a2a.png)

![IMG-002](https://user-images.githubusercontent.com/36792594/208579400-746bfb50-4096-444e-ae97-4fbd0944a0ec.png)

### flash_init

```bash
flash_init
```

![IMG-003](https://user-images.githubusercontent.com/36792594/208579403-48d8767d-4bb5-44f4-9ca6-8d846d3ea12d.png)


### Directory list

```bash
dir flash:
```

### Switch configuration 재설정

```bash
erase startup-config
# 또는
write erase
reload
```

### Delete config.text

```bash
delete flash:config.text
```

<details markdown="block">
  <summary>
    코드
  </summary>
  {: .text-delta }

```bash
switch: delete flash:config.text
Are you sure you want to delete "flash:config.text" (y/n)?y
File "flash:config.text" deleted
```

</details>
{: .label .label-green }

### Delete vlan.dat

```bash
delete flash:vlan.dat
```

<details markdown="block">
  <summary>
    코드
  </summary>
  {: .text-delta }

```bash
switch: del flash:vlan.dat
Are you sure you want to delete "vlan.dat" (y/n)?y
File "flash:vlan.dat" deleted
```

</details>
{: .label .label-green }

![IMG-004](https://user-images.githubusercontent.com/36792594/208579404-396308ed-189f-43c6-be5c-4ce959f3b736.png)

![IMG-005](https://user-images.githubusercontent.com/36792594/208579405-708b9bc9-d01b-43ff-b73d-78d26e610356.png)
![IMG-006](https://user-images.githubusercontent.com/36792594/208579406-3b8a050d-d631-40c0-ba54-3edc64e13665.png)
![IMG-007](https://user-images.githubusercontent.com/36792594/208579407-7bd183f6-c6d3-452f-8a8a-ecb22e2bfcf6.png)
![IMG-008](https://user-images.githubusercontent.com/36792594/208579409-1eda973e-aca5-43b3-8e4d-04face507c44.png)

![IMG-009](https://user-images.githubusercontent.com/36792594/208579411-b323e18a-915d-404a-b113-b2728c9ae16c.png)

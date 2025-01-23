---
title: Cisco Switch 복구 X/Y/Z MODEOM
author: G.G
date: 2025-01-08 10:56:00 +0900
categories: [Blog, Provisioning]
tags: [provisioning, cisco, x modeom, y modeom, z modeom, serial port]
---

## 개요
Cisco 스위치에서 **직렬 포트(Serial Port)**를 활용하여 XMODEM, YMODEM, ZMODEM과 같은 파일 전송 프로토콜로 Cisco IOS 이미지를 전송하는 방법은 네트워크 또는 USB 연결이 불가능한 경우에 사용됩니다. 이러한 방법은 로컬 관리 환경에서 스위치의 운영체제를 복구하거나 업그레이드하는 데 유용합니다. 특히 TFTP 서버를 사용할 수 없거나 네트워크 환경이 제한적인 경우에 활용됩니다.

## X / Y / Z MODEM 비교 표

| 기능                | XMODEM                        | YMODEM                             | ZMODEM                            |
|-------------------|-------------------------------|------------------------------------|-----------------------------------|
| 개발 연도            | 1977년                         | XMODEM 기반                         | YMODEM 기반                       |
| 블록 크기            | 128 바이트                     | 기본 1024 바이트, 128 바이트 지원      | 동적, 최대 4096 바이트              |
| 파일 이름/크기 전송  | 지원하지 않음                    | 지원                                | 지원                               |
| 전송 속도            | 느림                           | 중간                               | 빠름                               |
| 중단된 전송에서 이어받기 | 지원하지 않음                    | 지원 (YMODEM-G 제외)                | 지원                               |
| 배치 파일 전송       | 지원하지 않음                    | 지원                                | 지원                               |
| 오류 검출            | 체크섬 또는 CRC                  | CRC                                 | CRC                                |
| 비고                | 가장 기본적인 프로토콜            | XMODEM의 기능 향상 버전               | 전송 효율과 신뢰성이 높은 고급 기능 제공 |

## 사전 준비
1. 필요 장비
- Cisco 스위치
- USB-Serial 어댑터
- PC (Tera Term과 XMODEM 프로토콜 지원)
- 복구할 Cisco IOS 이미지 파일 (.bin)
2. 필요 소프트웨어
- Tera Term: 시리얼 통신 및 XMODEM 파일 전송 지원.
3. 시리얼 포트 설정 정보
- BAUD 속도: 기본 9600bps (필요 시 115200bps로 변경)
- Data Bits: 8
- Parity: None
- Stop Bits: 1
- Flow Control: None

## 복구 매뉴얼

### 스위치 연결
1. USB-Serial 어댑터를 통해 스위치의 콘솔 포트와 PC를 연결합니다.
2. Tera Term을 실행하고, Serial을 선택한 후 적절한 포트를 지정합니다 (예: COM11).

![Cisco_1](/assets/img/2025-01-13/Cisco-1.jpg)
_Tera Term Console COM11 선택_

![Cisco_2](/assets/img/2025-01-13/Cisco-2.jpg)
_Tera Term Console COM11 연결_

### BAUD 속도 설정
1.  스위치의 기본 BAUD 속도를 확인합니다.

```bash
switch: set
```

![Cisco_3](/assets/img/2025-01-13/Cisco-3.jpg)
_Tera Term Console set 확인_

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>
  
```bash
BAUD=9600
BOOT=flash:c2960-lanbasek9-mz.150-2.SE11.bin
CLEI_CODE_NUMBER=COM4A10BRB
MAC_ADDR=00:1C:F9:52:FD:00
MODEL_NUM=WS-C2960G-48TC-L
MODEL_REVISION_NUM=C0
MOTHERBOARD_ASSEMBLY_NUM=73-10300-07
MOTHERBOARD_REVISION_NUM=A0
MOTHERBOARD_SERIAL_NUM=FOC11292QVH
POWER_SUPPLY_PART_NUM=341-0098-02
POWER_SUPPLY_SERIAL_NUM=AZS112910S3
SDM_TEMPLATE_ID=0
SWITCH_PRIORITY=1
SYSTEM_SERIAL_NUM=FOC1129ZAJD
TAN_NUM=800-27071-02
TAN_REVISION_NUMBER=A0
VERSION_ID=V02
```

</details>

2. BAUD 속도를 더 빠른 115200bps로 변경합니다.

```bash
siwtch: set BAUD 115200
```

![Cisco_4](/assets/img/2025-01-13/Cisco-4.jpg)
_Tera Term Console set 설정_

3. Tera Term에서 시리얼 포트 설정을 변경합니다.

![Cisco_5](/assets/img/2025-01-13/Cisco-5.jpg)
_Tera Term Console 인코딩 깨짐 확인_

![TeraTerm_1](/assets/img/2025-01-08/TeraTerm_1.png)
_Tera Term Console Serial Port 선택_

![Cisco_6](/assets/img/2025-01-13/Cisco-6.jpg)
_Tera Term Console 인코딩 설정_

4. 변경 후 Tera Term을 다시 연결하여 설정이 적용되었는지 확인합니다.

```bash
switch: set
```

![Cisco_7](/assets/img/2025-01-13/Cisco-7.jpg)
_Tera Term Console 정상 출력 확인_

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>
  
```bash
BAUD=115200
BOOT=flash:c2960-lanbasek9-mz.150-2.SE11.bin
CLEI_CODE_NUMBER=COM4A10BRB
MAC_ADDR=00:1C:F9:52:FD:00
MODEL_NUM=WS-C2960G-48TC-L
MODEL_REVISION_NUM=C0
MOTHERBOARD_ASSEMBLY_NUM=73-10300-07
MOTHERBOARD_REVISION_NUM=A0
MOTHERBOARD_SERIAL_NUM=FOC11292QVH
POWER_SUPPLY_PART_NUM=341-0098-02
POWER_SUPPLY_SERIAL_NUM=AZS112910S3
SDM_TEMPLATE_ID=0
SWITCH_PRIORITY=1
SYSTEM_SERIAL_NUM=FOC1129ZAJD
TAN_NUM=800-27071-02
TAN_REVISION_NUMBER=A0
VERSION_ID=V02
```

</details>

### 복구 모드에서 XMODEM 전송 준비
1. 스위치에서 복구 명령을 입력하여 XMODEM 전송 준비를 활성화합니다.

```bash
copy xmodem: flash:c2960-lanbasek9-mz.150-2.SE11.bin
```

![Cisco_8](/assets/img/2025-01-13/Cisco-8.jpg)
_Tera Term Console XMODEM 전송 준비_

### Tera Term에서 XMODEM 파일 전송

### Serial Port 설정
1. Tera Term 상단 메뉴에서 File > Transfer > XMODEM > Send를 선택합니다.
2. 복구할 Cisco IOS 이미지 파일을 선택합니다 (예: c2960-lanbasek9-mz.150-2.SE11.bin).
3. 전송 진행 상황 창이 나타나며 파일이 전송됩니다.
- 전송 속도는 BAUD 속도에 따라 달라질 수 있습니다.
- BAUD 속도가 9600bps라면 전송 속도가 느리기 때문에 115200bps 설정이 추천됩니다.

![TeraTerm_3](/assets/img/2025-01-08/TeraTerm_3.png)
_Tera Term Console Send 선택_

![Cisco_9](/assets/img/2025-01-13/Cisco-9.jpg)
_Tera Term Console bin 파일 선택_

### 이미지 전송 완료

![Cisco_10](/assets/img/2025-01-13/Cisco-10.jpg)
_Tera Term Console XMODEM 전송 완료_

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>
  
```bash
Begin the Xmodem or Xmodem-1K transfer now...
C.........................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................
File "xmodem:" successfully copied to "flash:c2960-lanbasek9-mz.150-2.SE11.bin"
```

</details>

![Cisco_11](/assets/img/2025-01-13/Cisco-11.jpg)
_Tera Term Console XMODEM 전송 완료_

1. 복구된 이미지를 기본 부팅 이미지로 설정합니다.

```bash
set BOOT flash:c2960-lanbasek9-mz.150-2.SE11.bin
```

2. 복구된 이미지를 기본 부팅 이미지로 스위치를 재부팅 합니다.

```bash
boot
```

3. 복구된 이미지를 기본 부팅 이미지로 설정합니다.

```bash
configure terminal
boot system flash:c2960-lanbasek9-mz.150-2.SE11.bin
```

4. 변경 사항 저장합니다.

```bash
write memory
```

## 참조
- [TeraTerm 도구 다운로드](https://github.com/TeraTermProject/teraterm/releases)

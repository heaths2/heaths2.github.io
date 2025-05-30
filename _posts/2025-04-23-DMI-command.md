---
title: DMI table decoder 명령어 사용법
author: G.G
date: 2025-04-23 11:03:00 +0900
categories: [Blog, Command]
tags: [Command, Script, dmidecode]
---

## ⚙️ 명령어 사용법

### dmidecode Type 번호 및 키워드 매핑표

| Keyword    | Types                  |
|------------|------------------------|
| bios       | 0, 13                  |
| system     | 1, 12, 15, 23, 32      |
| baseboard  | 2, 10, 41              |
| chassis    | 3                      |
| processor  | 4                      |
| memory     | 5, 6, 16, 17           |
| cache      | 7                      |
| connector  | 8                      |
| slot       | 9                      |


```bash
# 메모리 모듈 정보 확인 (type 17: Memory Device)
# 크기(size), 슬롯 위치(locator), memory 관련 키워드 포함 항목만 출력
dmidecode --type 17 | egrep --ignore-case 'memory|size|locator'

# 위와 동일한 내용, 'memory' 키워드로도 같은 결과 (type 17 포함)
dmidecode --type memory | egrep -i 'memory|size|locator'


# CPU(프로세서) 정보 확인 (type 4: Processor Information)
# 소켓 이름(Slot 위치), CPU 버전 정보 확인
dmidecode --type 4 | egrep -i 'Socket Designation|version'

# 위와 동일한 내용, 'processor' 키워드로도 같은 결과
dmidecode --type processor | egrep -i 'Socket Designation|version'


# CPU의 코어 수와 스레드 수 확인 (Core Count, Thread Count)
dmidecode -t 4 | grep -E 'Core Count|Thread Count'


# 확장 슬롯(PCIe 등) 정보 확인 (type 9: System Slot)
# 슬롯 이름, 타입, 현재 사용 상태 출력
dmidecode -t 9 | grep -Ei 'Designation|Type|Current Usage'

# 위와 동일한 내용, 'slot' 키워드로도 같은 결과
dmidecode -t slot | grep -Ei 'Designation|Type|Current Usage'
```

```bash
#!/usr/bin/env bash 
set -eE 
#export LC_ALL=ko_KR.UTF-8 
export LC_ALL=en_US.UTF-8

# 로그 파일 정의 
log_file="./runLog-$(date +'%F_%H%M%S').log"

# 모든 출력을 로그 파일과 콘솔에 동시에 기록 
exec &> >(tee -a "$log_file")
 
echo "\
OS :            $(lsb_release -is)
Relase :        $(lsb_release -rs)
Code Name :     $(lsb_release -cs)
Kernel :        $(uname -r)
Architecture :  $(uname -m)
Manufacturer :  $(dmidecode -t 1 | grep 'Manufacturer:' | awk '{ print $NF }')
Product Name :  $(dmidecode -t 1 | grep 'Product Name:' | awk '{ print $NF }')
CPU Core :      $(dmidecode -t 4 | grep "Thread Count:" | awk '{ sum += $3 } END { print sum }')
Memory :        $(dmidecode -t 17 | grep Size: | grep -v No | sed 's/[^0-9]//g' | paste -sd+ | bc
)" > Result
```

## 참고 자료
> - [DMI table decoder 참고 사이트](https://linux.die.net/man/8/dmidecode/)
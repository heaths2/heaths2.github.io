---
title: IPtables 사용법
author: G.G
date: 2025-05-07 21:48 +0900
categories: [Blog, Command]
tags: [Command, iptables]
---

## 📌 개요
이 문서는 리눅스 서버 보안을 위해 iptables를 사용하여 기본 방화벽 정책을 구성하는 방법을 설명합니다.

특히 SSH 접속과 ICMP(ping) 요청만 허용하고, 나머지 트래픽은 차단하는 화이트리스트 방식의 최소 보안 정책을 중심으로 구성되어 있습니다.

이 문서에서는 iptables 명령어의 주요 옵션 설명부터 시작해, 실제 적용 가능한 규칙 설정, 저장 및 복원, 정책 제거 방법까지 실무에 바로 적용할 수 있도록 예시와 함께 정리되어 있습니다.


## 🧩 정책 설명

| **정책 방식**        | **기본 정책**       | **트래픽 처리 방식**                     | **설정 명령어**                                                                 |
|-----------------------|--------------------|------------------------------------------|-------------------------------------------------------------------------------|
| **화이트리스트**      | `DROP` (차단)      | 허용할 트래픽만 명시적으로 `ACCEPT`      | `sudo iptables -P INPUT DROP`<br>`sudo iptables -P FORWARD DROP`<br>`sudo iptables -P OUTPUT DROP`<br> |
| **블랙리스트**        | `ACCEPT` (허용)    | 차단할 트래픽만 명시적으로 `DROP`        | `sudo iptables -P INPUT ACCEPT`<br>`sudo iptables -P FORWARD ACCEPT`<br>`sudo iptables -P OUTPUT ACCEPT`<br> |

| **기준**                 | **화이트리스트**                | **블랙리스트**              |
|--------------------------|---------------------------------|----------------------------|
| **보안 수준**            | 높음                           | 낮음                       |
| **설정 복잡도**          | 상대적으로 복잡                | 간단                       |
| **트래픽 처리**          | 명시적으로 허용된 것만 통과     | 명시적으로 차단된 것만 막음 |
| **추천 환경**            | 보안이 중요한 환경             | 보안이 덜 중요한 환경      |

- `iptables` 옵션

| 명령어 (단일 `-`) | 명령어 (이중 `--`)  | 설명                                                | 사용 예시                                      |
|:------------------|:---------------------|:----------------------------------------------------|:-----------------------------------------------|
| -A               | --append            | 체인의 끝에 규칙을 추가합니다.                      | iptables -A INPUT -s 192.168.0.1 -j ACCEPT    |
| -D               | --delete            | 체인의 규칙을 삭제합니다.                          | iptables -D INPUT 1                           |
| -I               | --insert            | 체인의 특정 위치에 규칙을 삽입합니다.               | iptables -I INPUT 1 -s 192.168.0.1 -j ACCEPT  |
| -R               | --replace           | 체인의 특정 위치에 있는 규칙을 대체합니다.          | iptables -R INPUT 1 -s 192.168.0.1 -j ACCEPT  |
| -L               | --list              | 체인의 규칙 목록을 출력합니다.                      | iptables -L                                   |
| -F               | --flush             | 체인의 모든 규칙을 삭제합니다.                      | iptables -F                                   |
| -X               | --delete-chain      | 모든 사용자 정의 체인을 삭제합니다.                 | iptables -X                                   |
| -P               | --policy            | 체인의 기본 정책을 설정합니다.                      | iptables -P INPUT DROP                        |
| -N               | --new-chain         | 새로운 사용자 정의 체인을 생성합니다.               | iptables -N my_chain                          |
| -E               | --rename-chain      | 사용자 정의 체인의 이름을 변경합니다.               | iptables -E old_chain new_chain               |
| -p               | --protocol          | 패킷의 프로토콜을 지정합니다.                       | iptables -A INPUT -p tcp                      |
| -s               | --source            | 소스 IP 주소를 지정합니다.                          | iptables -A INPUT -s 192.168.0.1              |
| -d               | --destination       | 목적지 IP 주소를 지정합니다.                        | iptables -A INPUT -d 192.168.0.1              |
|                  | --dport             | 목적지 포트를 지정합니다.                           | iptables -A INPUT -p tcp --dport 80           |
|                  | --sport             | 소스 포트를 지정합니다.                             | iptables -A INPUT -p tcp --sport 1024         |
| -i               | --in-interface      | 입력 인터페이스를 지정합니다.                       | iptables -A INPUT -i eth0                     |
| -o               | --out-interface     | 출력 인터페이스를 지정합니다.                       | iptables -A OUTPUT -o eth0                    |
| -m state         |                     | 상태 일치 모듈을 사용하여 패킷 상태를 지정합니다.   | iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT |
|                  | --state             | 패킷의 상태를 지정합니다 (NEW, ESTABLISHED, RELATED, INVALID). | iptables -A INPUT -m state --state NEW        |
| -j               | --jump              | 패킷이 일치할 경우 실행할 타겟을 지정합니다 (ACCEPT, DROP 등). | iptables -A INPUT -j ACCEPT                   |
|                  | --icmp-type         | ICMP 패킷의 유형을 지정합니다.                       | iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT |
| -n               | --numeric           | IP 주소와 포트를 숫자 형식으로 출력합니다.           | iptables -L -n                                |
| -v               | --verbose           | 자세한 정보를 출력합니다.                            | iptables -L -v                                |
|                  | --line-numbers      | 각 규칙의 줄 번호를 표시합니다.                      | iptables -L --line-numbers                    |

- 방화벽 정책

| Policy ID | From      | To        | SIP           | SPORT    | DIP      | DPORT | SERVICE | Schedule  | Action    | NAT         | Logging     |
|:----------|:----------|:----------|:--------------|:---------|:---------|:------|:--------|:----------|:----------|:------------|:------------|
| 1         | Internal  | External  | all           | ALL      | all      | ALL   | ALL     | ⏰ always | ✅ ACCEPT | ❌ Disabled | ✅ ALL      |
| 2         | External  | Internal  | all           | ALL      | all      | 53<br>123 | DNS<br>NTP | ⏰ always | ✅ ACCEPT | ❌ Disabled | ✅ ALL      |
| 3         | External  | Internal  | all           | ALL      | 172.16.0.0/24 | 22   | SSH<br>PING | ⏰ always | ✅ ACCEPT | ❌ Disabled | ✅ ALL      |
| 4         | External  | Internal  | all           | ALL      | 10.1.0.0/24   | 22   | SSH     | ⏰ always | ✅ ACCEPT | ❌ Disabled | ✅ ALL      |
| 0         | any       | any       | all           | ALL      | all      | ALL   | ALL     | ⏰ always | 🚫 DENY   | ❌ Disabled | ❌ Disabled |

- iptables 규칙 목록 출력

```bash
sudo iptables -L --line-numbers -n -v
```

## 🏗️ 정책 설정 내용용

```bash
Chain Chain INPUT (policy DROP 244 packets, 33556 bytes)
num    pkts bytes target     prot opt in     out     source               destination         
1        68  7408 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
2      1413  114K ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
3         0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp spt:53
4         0     0 ACCEPT     udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp spt:53
5         0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp spt:123
6         0     0 ACCEPT     udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp spt:123
7         1    64 ACCEPT     tcp  --  *      *       172.16.0.0/24        0.0.0.0/0            tcp dpt:22
8         0     0 ACCEPT     tcp  --  *      *       10.1.0.0/24          0.0.0.0/0            tcp dpt:22
9         0     0 ACCEPT     icmp --  *      *       172.16.0.0/24        0.0.0.0/0            icmptype 8
10        0     0 LOG        tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22 LOG flags 0 level 7 prefix "iptables_SSH_accept: "
11        0     0 LOG        tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22 LOG flags 0 level 7 prefix "iptables_SSH_deny: "
12      144 12526 LOG        all  --  *      *       0.0.0.0/0            0.0.0.0/0            LOG flags 0 level 7 prefix "iptables_INPUT_denied: "      

Chain Chain FORWARD (policy DROP 0 packets, 0 bytes)
num    pkts bytes target     prot opt in     out     source               destination         
1         0     0 LOG        all  --  *      *       0.0.0.0/0            0.0.0.0/0            LOG flags 0 level 7 prefix "iptables_FORWARD_denied: "
2         0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain Chain OUTPUT (policy DROP 180 packets, 33234 bytes)
num    pkts bytes target     prot opt in     out     source               destination         
1        68  7408 ACCEPT     all  --  *      lo      0.0.0.0/0            0.0.0.0/0           
2       570 42607 ACCEPT     all  --  *      eth0    0.0.0.0/0            0.0.0.0/0           
3       695  329K ACCEPT     all  --  *      eth1    0.0.0.0/0            0.0.0.0/0           
4         0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
5         0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:53
6         0     0 ACCEPT     udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:53
7         0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:123
8         0     0 ACCEPT     udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:123
9         0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            172.16.0.0/24        tcp spt:22
10        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            10.1.0.0/24          tcp spt:22
11        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            172.16.0.0/24        icmptype 0
12        0     0 LOG        tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp spt:22 LOG flags 0 level 7 prefix "iptables_SSH_accept: "
13        0     0 LOG        tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp spt:22 LOG flags 0 level 7 prefix "iptables_SSH_deny: "
14       32  1920 LOG        all  --  *      *       0.0.0.0/0            0.0.0.0/0            LOG flags 0 level 7 prefix "iptables_OUTPUT_denied: "
```

## ⚙️ 사용법

```bash
cat <<'EOF' | sudo tee /etc/rc.firewall
#!/usr/bin/bash

# 모든 규칙 플러시
sudo iptables -F

# 사용자 정의 체인 삭제
sudo iptables -X

# 모든 허용되지 않은 트래픽 기본 정책으로 차단
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT DROP

# 루프백 인터페이스의 트래픽 허용
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# eth0 인터페이스의 트래픽 허용
sudo iptables -A OUTPUT -o eth0 -j ACCEPT

# eth1 인터페이스의 트래픽 허용
sudo iptables -A OUTPUT -o eth1 -j ACCEPT

# 이미 확립된 연결 및 관련 트래픽 허용
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# eth0 → eth1으로 트래픽 허용
#sudo iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT

# eth1 → eth0으로 트래픽 허용
#sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT

# ICMP(PING) 트래픽 허용
#sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
#sudo iptables -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT

# DNS 트래픽 허용 (TCP/UDP 포트 53)
sudo iptables -A OUTPUT -p tcp --dport 53 -j ACCEPT
sudo iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p tcp --sport 53 -j ACCEPT
sudo iptables -A INPUT -p udp --sport 53 -j ACCEPT

# NTP 트래픽 허용 (TCP/UDP 포트 123)
sudo iptables -A OUTPUT -p tcp --dport 123 -j ACCEPT
sudo iptables -A OUTPUT -p udp --dport 123 -j ACCEPT
sudo iptables -A INPUT -p tcp --sport 123 -j ACCEPT
sudo iptables -A INPUT -p udp --sport 123 -j ACCEPT

# SSH 접근 시도 로깅 (허용된 경우)
# sudo iptables -A INPUT -p tcp --dport 22 -j LOG --log-prefix "iptables_INPUT_SSH_Accept: " --log-level 7
# sudo iptables -A OUTPUT -p tcp --sport 22 -j LOG --log-prefix "iptables_OUTPUT_SSH_Accept: " --log-level 7

# SSH(포트 22) 트래픽 허용
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT
# sudo iptables -A INPUT -s 172.16.0.0/24 -p tcp --dport 22 -j ACCEPT
# sudo iptables -A INPUT -s 10.1.0.0/24 -p tcp --dport 22 -j ACCEPT
# sudo iptables -A INPUT -s 172.16.0.0/24 -p icmp --icmp-type echo-request -j ACCEPT
# sudo iptables -A INPUT -s 10.1.0.0/24 -p icmp --icmp-type echo-request -j ACCEPT
# sudo iptables -A OUTPUT -d 172.16.0.0/24 -p tcp --sport 22 -j ACCEPT
# sudo iptables -A OUTPUT -d 10.1.0.0/24 -p tcp --sport 22 -j ACCEPT
# sudo iptables -A OUTPUT -d 172.16.0.0/24 -p icmp --icmp-type echo-reply -j ACCEPT
# sudo iptables -A OUTPUT -d 10.1.0.0/24 -p icmp --icmp-type echo-reply -j ACCEPT

# SSH 접근 시도 로깅 (차단된 경우)
# sudo iptables -A INPUT -p tcp --dport 22 -j LOG --log-prefix "iptables_INPUT_SSH_Deny: " --log-level 7
# sudo iptables -A OUTPUT -p tcp --sport 22 -j LOG --log-prefix "iptables_OUTPUT_SSH_Deny: " --log-level 7

# 모든 허용되지 않은 트래픽 로깅
sudo iptables -A INPUT -j LOG --log-prefix "iptables_INPUT_Denied: " --log-level 7
sudo iptables -A FORWARD -j LOG --log-prefix "iptables_FORWARD_Denied: " --log-level 7
sudo iptables -A OUTPUT -j LOG --log-prefix "iptables_OUTPUT_Denied: " --log-level 7
EOF
```

### iptables 특정 규칙 제거

```bash
sudo iptables -D INPUT 13
sudo iptables -D FORWARD 2
sudo iptables -D OUTPUT 15
```

## iptables 저장 및 복원 명령어

### iptables 저장
- **iptables 규칙 저장**

현재 설정된 iptables 규칙을 파일에 저장

```bash
iptables-save > iptables.rules
```

### iptables 복원
- **iptables 규칙 복원**

`iptables.rules` 파일에 저장된 iptables 규칙을 복원

```bash
iptables-restore < iptables.rules
```
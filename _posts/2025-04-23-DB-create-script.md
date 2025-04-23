---
title: Database 명령어 사용법
author: G.G
date: 2025-04-23 09:36:00 +0900
categories: [Blog, Command]
tags: [Command, Script, MySQL, MariaDB]
---

## 명령어 사용법

```bash
-- 현재 MySQL 서버에 존재하는 데이터베이스 목록 보기
SHOW DATABASES;

-- mysql이라는 시스템 데이터베이스 사용 (계정, 권한 등 정보 저장됨)
USE mysql;

-- 현재 선택한 데이터베이스의 테이블 목록 보기
SHOW TABLES;

-- 'awx'라는 이름의 데이터베이스 생성 (UTF-8 문자셋 설정)
CREATE DATABASE awx CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 'awx'라는 이름의 사용자 생성, 비밀번호는 'awx'
CREATE USER awx IDENTIFIED BY 'awx';

-- 'awx' 사용자에게 'awx' 데이터베이스에 대한 모든 권한 부여
GRANT ALL PRIVILEGES ON `awx`.* TO 'awx'@'localhost';

-- 권한 변경사항을 즉시 적용
FLUSH PRIVILEGES;

-- mysql.user 테이블에서 등록된 사용자와 접근 가능한 호스트 정보 조회
SELECT Host,User FROM user;

-- 'awx'@'localhost' 사용자가 가지고 있는 권한 목록 확인
SHOW GRANTS FOR 'awx'@'localhost';

-- 'awx' 데이터베이스 삭제 (주의: 데이터도 함께 삭제됨)
DROP DATABASE awx;

-- 'awx'@'localhost' 사용자 삭제 (주의: 사용자 연결도 삭제됨)
DROP USER 'awx'@'localhost';

-- 최대 패킷 크기 확인 (기본 16MB 또는 64MB가 일반적)
SHOW VARIABLES LIKE 'max_allowed_packet';

-- 현재 접속한 클라이언트/서버의 실행 중인 프로세스 보기 (복제 쓰레드 포함)
SHOW PROCESSLIST;

-- 이 서버가 슬레이브(Replica)인지 확인
SHOW SLAVE STATUS\G

-- 현재 이 서버의 마스터에 연결된 슬레이브 목록 보기 (마스터에서 실행)
SHOW SLAVE HOSTS;

-- 특정 스키마(데이터베이스)의 문자셋 및 콜레이션(정렬 방식) 확인
SELECT 
    SCHEMA_NAME, 
    DEFAULT_CHARACTER_SET_NAME, 
    DEFAULT_COLLATION_NAME 
FROM 
    information_schema.SCHEMATA
WHERE 
    SCHEMA_NAME = 'awx';

-- 특정 스키마(데이터베이스)의 문자셋 및 콜레이션(정렬 방식) 확인
SHOW CREATE DATABASE awx;
```

```bash
#!/usr/bin/env bash
# ------------------------------------------------------------------------------
# MySQL 데이터베이스 및 사용자 초기화 스크립트
# 목적: UTF-8 설정으로 데이터베이스 생성 + 사용자 권한 설정
# 사용 환경: AWX 또는 기타 어플리케이션 초기 DB 셋업 시
# ------------------------------------------------------------------------------

# ------------------------------------------------------------------------------
# 설정값 (보안상 실제 배포시에는 환경변수 또는 Vault 사용 권장)
# ------------------------------------------------------------------------------
DB_NAME="awx"
DB_USER="awx"
DB_PASS="awx"

# ------------------------------------------------------------------------------
# 루트 계정으로 MySQL 명령어 실행
# - 데이터베이스 생성
# - 사용자 생성 및 권한 부여
# - 적용된 사용자 목록 및 DB 목록 확인
# ------------------------------------------------------------------------------
mysql -u root -e "
    -- [1] 데이터베이스가 존재하지 않으면 생성
    CREATE DATABASE IF NOT EXISTS \`$DB_NAME\`
        CHARACTER SET utf8mb4
        COLLATE utf8mb4_unicode_ci;

    -- [2] 사용자 계정이 존재하지 않으면 생성 (로컬 접속 전용)
    CREATE USER IF NOT EXISTS '$DB_USER'@'localhost'
        IDENTIFIED BY '$DB_PASS';

    -- [3] 해당 DB에 대해 모든 권한 부여
    GRANT ALL PRIVILEGES ON \`$DB_NAME\`.* TO '$DB_USER'@'localhost';

    -- [4] 권한 설정 반영
    FLUSH PRIVILEGES;

    -- [5] 디버깅용: 사용자 목록 및 DB 목록 출력
    SELECT User, Host FROM mysql.user;
    SHOW DATABASES;
"
```


## 참조
- [증권정보포털 세이브로](https://seibro.or.kr)

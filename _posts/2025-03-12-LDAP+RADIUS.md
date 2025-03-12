---
title:  LDAP + RADIUS 설치 매뉴얼
author: G.G
date: 2025-03-12 13:11:00 +0900
categories: [Blog, Provisioning]
tags: [provisioning, ldap, radius]
---

```bash
#!/usr/bin/env bash
set -eE
export LC_ALL=ko_KR.UTF-8

# 로그 파일 정의
log_file="/tmp/$(basename "$0")-$(date +'%F_%H%M%S').log"

# 모든 출력을 로그 파일과 콘솔에 동시에 기록
exec &> >(tee -a "$log_file")

title() {
    echo "----- $1 -----"
}

title "필수 패키지 설치"
apt-get update
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh | printf "\n" | read
sudo apt-get install -y freeradius freeradius-ldap freeradius-postgresql postgresql libpam-google-authenticator
title "필수 패키지 설치 완료"

title "PostgreSQL 설정 시작"
sudo -u postgres psql -c "
-- PostgreSQL 사용자 생성 및 비밀번호 설정
CREATE USER radius WITH PASSWORD 'qwer1234';
"
sudo -u postgres psql -c "
-- PostgreSQL 데이터베이스 생성
CREATE DATABASE radius;
"

# FreeRADIUS 스키마를 PostgreSQL 데이터베이스에 로드
sudo -u postgres psql radius < /etc/freeradius/3.0/mods-config/sql/main/postgresql/schema.sql

# 권한 부여 스크립트 및 해시 업데이트
sudo -u postgres psql -d radius -c "
-- public 스키마의 모든 테이블에 대해 권한 부여
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO radius;

-- 앞으로 생성될 모든 테이블에 대해 권한 부여
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO radius;

-- 모든 시퀀스에 대해 권한 부여
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO radius;

-- 앞으로 생성될 모든 시퀀스에 대해 권한 부여
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO radius;

-- pgcrypto 확장 생성
CREATE EXTENSION pgcrypto;
"

sudo -u postgres psql -d radius -c "INSERT INTO radcheck (username, attribute, op, value) VALUES ('bob', 'SSHA2-512-Password', ':=', '{ssha512}zyY48qKTPV48enAwlcZAtPxC3KzonKSlr2d37yd0N1CGYxyt4wNHKP7EITgokf8Bz5H51zJ90EzcSZK3ThiM4s0695ZKKeZ0xXGaqq3BSbg=');"
sudo -u postgres psql -d radius -c "INSERT INTO radusergroup (username, groupname, priority) VALUES ('bob', 'RadiusUsers', 1);"

sudo -u postgres psql -d radius -c "SELECT * FROM radcheck;"

sudo -u postgres psql -d radius -c "SELECT * FROM radusergroup;"
title "PostgreSQL 설정 완료"

title "FreeRADIUS 설정"
sed -i -e '349s/auth = no/auth = yes/' \
       -e '373s/auth_badpass = no/auth_badpass = yes/' \
       -e '374s/auth_goodpass = no/auth_goodpass = yes/' /etc/freeradius/3.0/radiusd.conf
sed -n -e '349p' \
       -e '373p' \
       -e '374p' /etc/freeradius/3.0/radiusd.conf

cd /etc/freeradius/3.0/mods-enabled
ln -s ../mods-available/sql sql
chown -h $(id -u freerad):$(id -g freerad) sql

sed -i -e 's/dialect = "sqlite"/dialect = "postgresql"/' \
       -e 's/driver = "rlm_sql_null"/#&/' \
       -e '/driver = "rlm_sql_${dialect}"/ s/^#//' \
       -e '/server = "localhost"/ s/^#//' \
       -e '/port = 3306/ s/^#//' \
       -e 's/port = 3306/port = 5432/' \
       -e '/login = "radius"/ s/^#//' \
       -e '/read_clients = yes/ s/^#//' \
       -e '/password = "radpass"/ s/^#//' \
       -e 's/password = "radpass"/password = "qwer1234"/' /etc/freeradius/3.0//mods-available/sql
title "FreeRADIUS 설정 완료"

title "PAM 설정"
cd /etc/freeradius/3.0/mods-enabled
ln -s ../mods-available/pam pam
chown -h $(id -u freerad):$(id -g freerad) pam

sed -i -e 's/-sql/sql/' \
       -e 's/-ldap/ldap/' /etc/freeradius/3.0/sites-available/inner-tunnel

sed -i -e '566,568s/^#//' \
       -e 's/-ldap/ldap/' \
       -e '/sql/ s/-//' \
       -e '/pam/ s/^#//' \
       -e '/auth_log/ s/^#//' \
       -e '/reply_log/ s/^#//' /etc/freeradius/3.0/sites-available/default

sed -i 's/@include/#@include/g' /etc/pam.d/radiusd

echo "
# Google Authenticator 모듈을 사용한 OTP 인증
auth       required     pam_google_authenticator.so forward_pass secret=/etc/freeradius/3.0/pam.d/\${USER}/.google_authenticator user=freerad
# 시스템 사용자 계정 인증
auth       required    pam_permit.so use_first_pass\
" >> /etc/pam.d/radiusd

cat /etc/pam.d/radiusd

echo "
# 기본 정책 설정: 모든 사용자는 PAM 인증을 사용
DEFAULT Auth-Type := PAM\
" >> /etc/freeradius/3.0/mods-config/files/authorize
sed -n '207,$p' /etc/freeradius/3.0/mods-config/files/authorize

querys=$(echo "\
        # SHA512 해시 적용 쿼리 추가\n\
        query = \"\\\\\n\
                INSERT INTO \${..postauth_table} \\\\\n\
                        (username, pass, reply, calledstationid, callingstationid, authdate \${..class.column_name}) \\\\\n\
                VALUES(\\\\\n\
                        '%{User-Name}', \\\\\n\
                        digest('%{%{User-Password}:-%{Chap-Password}}', 'sha512'), \\\\\n\
                        '%{reply:Packet-Type}', \\\\\n\
                        '%{Called-Station-Id}', \\\\\n\
                        '%{Calling-Station-Id}', \\\\\n\
                        '%S.%M' \\\\\n\
                        \${..class.reply_xlat})\"")

sed -i -e '690,698s/^/#/' \
       -e "698a\\${querys}" /etc/freeradius/3.0/mods-config/sql/main/postgresql/queries.conf
title "PAM 설정 완료"

title "클라이언트 설정"
echo "
client FortiGate-60E {
        ipaddr          = 10.1.0.0
        netmask         = 16
        port            = 1812
        proto           = *
        secret          = testing123
        require_message_authenticator = no
        nas_type        = other
}" >> /etc/freeradius/3.0/clients.conf
title "클라이언트 설정 완료"

title "ReadOnlyDirectories 설정 제거"
sed -i 's/ReadOnlyDirectories=\/etc\/freeradius/#ReadOnlyDirectories=\/etc\/freeradius/' /lib/systemd/system/freeradius.service
title "ReadOnlyDirectories 설정 제거 완료"

title "서비스 재시작"
systemctl daemon-reload &&\
systemctl enable --now freeradius.service
title "서비스 재시작 완료"

echo "
# 아래 명령어를 사용하여 bob 2FA 인증키를 발급
(
       echo "-1"
       yes n
) | \
su ${user:-bob} -c 'google-authenticator -tdf -r 1 -Q UTF8 -i G.G -w 3 -R 30 -s /home/${user:-bob}/.ssh/.google_authenticator'

# Google Authenticator 2Factor 복사
mkdir -p /etc/freeradius/3.0/pam.d/bob
cd /etc/freeradius/3.0/pam.d/bob
cp -av /home/bob/.ssh/.google_authenticator .google_authenticator
chown -R freerad:freerad /etc/freeradius/3.0/pam.d/

# 시스템 유저 생성
username="bob"
password="qwer1234"
useradd -m "$username" -d "/home/$username" -p "$(openssl passwd -6 "$password")" -s /bin/bash -c "$username"

# 테스트 방법
# FreeRADIUS 디버깅 활성화를 하려면 아래 명령어를 입력
freeradius -X
radtest bob qwer1234+OTP localhost 0 testing123

sudo -u postgres psql -d radius -c '
-- radpostauth 테이블의 모든 데이터 선택
SELECT * FROM radpostauth ORDER BY id DESC;'

# 로그 확인
tail -f /var/log/auth
"
```

```bash
sed -i.bak -e "s/server = 'localhost'/server = '$(hostname -I | awk '{print $1}')'/" \
           -e "28,29s/^#//" \
           -e "s/identity = .*/identity = 'cn=admin,dc=infra,dc=com'/" \
           -e "s/password = .*/password = '1234'/" /etc/freeradius/3.0/mods-available/ldap
```

---
title: PowerDNS-Admin 설치방법
author: G.G
date: 2025-01-03 20:50:00 +0900
categories: [Blog, Provisioning]
tags: [powerdns, powerdns-admin, bind9]
---

## 개요
PowerDNS-Admin은 PowerDNS 서버의 관리 인터페이스를 제공하는 웹 애플리케이션입니다. 이 도구는 PowerDNS의 설정을 직관적인 웹 UI를 통해 관리할 수 있게 해줍니다. PowerDNS-Admin은 사용자가 DNS 레코드를 쉽게 추가, 수정, 삭제할 수 있도록 도와주며, 여러 도메인과 DNS 설정을 관리할 수 있습니다.

## 특징
1. 직관적인 웹 UI
- PowerDNS-Admin은 PowerDNS 서버의 설정을 시각적이고 직관적인 웹 인터페이스로 관리할 수 있게 합니다.

2. 다중 사용자 지원:
- 여러 사용자 계정을 관리할 수 있어 다양한 권한 설정이 가능합니다.

3. 권한 기반 액세스 제어
- 특정 도메인이나 레코드에 대한 접근을 제한할 수 있는 권한 관리 기능이 포함되어 있습니다.

4. SQLite/PostgreSQL/MySQL 지원
- 데이터베이스로 SQLite, PostgreSQL, MySQL을 지원하여 다양한 환경에 맞춰 사용할 수 있습니다.

5. 로그인 기능
- 인증된 사용자만 PowerDNS 서버에 접근할 수 있도록 로그인 기능을 제공합니다.

6. API 지원
- RESTful API를 통해 자동화된 DNS 관리가 가능합니다.

## 구성요소
1. PowerDNS
- DNS 서비스를 제공하는 주요 서버. PowerDNS-Admin은 이를 관리하는 웹 애플리케이션입니다.

2. PowerDNS-Admin
- PowerDNS의 DNS 레코드와 설정을 관리하는 웹 애플리케이션.

3. 데이터베이스
- PowerDNS-Admin은 DNS 데이터와 사용자 정보를 저장하기 위해 SQLite, PostgreSQL, 또는 MySQL을 사용할 수 있습니다.

4. 웹 서버
- PowerDNS-Admin은 Flask 기반의 웹 애플리케이션으로, 웹 서버가 필요합니다. Apache 또는 Nginx와 같은 서버에서 호스팅할 수 있습니다.

5. 인증 및 보안
- 사용자 인증을 위해 필요한 보안 설정이 필요합니다 (예: SSL 설정).

## 환경구성
1. 로케일 설정
2. 패키지 설치

```bash
localectl set-locale LANG=en_US.UTF-8
```

```bash
sudo apt-get update
sudo apt-get install software-properties-common gnupg2 lsb-release curl
```

## 설치

### PostgreSQL
1. PostgreSQL 설치
2. PostgreSQL 서버 서비스 등록
3. `/opt/pdns_install/db_credentials`{: .filepath} 환경변수 파일 생성
```bash
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo apt update
sudo apt -y install postgresql
```

```bash
systemctl enable --now postgresql
```

```bash
#!/usr/bin/env bash

# Create a Working Directory for the installation
mkdir -p /opt/pdns_install
workpath="/opt/pdns_install"

pdns_db="pdns"
pdns_db_user="pdns"
pdns_pwd="$(tr -dc A-Za-z0-9 </dev/urandom | head -c 13 ; echo '')"
pdnsadmin_salt="$(tr -dc A-Za-z0-9 </dev/urandom | head -c 32 ; echo '')"
pdns_apikey="$(tr -dc A-Za-z0-9 </dev/urandom | head -c 32 ; echo '')"

echo "pdns_db=$pdns_db" > "$workpath/db_credentials"
echo "pdns_db_user=$pdns_db_user" >> "$workpath/db_credentials"
echo "pdns_pwd=$pdns_pwd" >> "$workpath/db_credentials"
echo "pdnsadmin_salt=$pdnsadmin_salt" >> "$workpath/db_credentials"
echo "pdns_apikey=$pdns_apikey" >> "$workpath/db_credentials"
echo "workpath=$workpath" >> "$workpath/db_credentials"
echo 'systemReleaseVersion='$(lsb_release -cs) >> "$workpath/db_credentials"
echo 'osLabel='$(lsb_release -sa 2>/dev/null|head -n 1|tr '[:upper:]' '[:lower:]') >> "$workpath/db_credentials"
chown root:root "$workpath/db_credentials"
chmod 640 "$workpath/db_credentials"
cd "$workpath"

# PostgreSQL - Set this variable for stuff down the road
db_type="pgsql"
echo "db_type=$db_type" >> "$workpath/db_credentials"

cat /opt/pdns_install/db_credentials
```

```bash
echo \
"-- PowerDNS PGSQL Create DB File
CREATE USER pdns WITH ENCRYPTED PASSWORD '$pdns_pwd';
CREATE DATABASE pdns OWNER pdns;
GRANT ALL PRIVILEGES ON DATABASE pdns TO pdns;" > "/opt/pdns_install/pdns-createdb-pg.sql"
sudo -u postgres psql < "/opt/pdns_install/pdns-createdb-pg.sql"
```

### PowerDNS
1. PowerDNS 저장소 추가 및 Pinning 설정
2. 저장소 서명 키 추가
3. 시스템 DNS Resolver 비활성화 및 기 DNS 설정
4. PowerDNS 설치 및 PostgreSQL 데이터베이스 PowerDNS 모듈 설치
5. PostgreSQL 데이터베이스 데이터 넣기
6. PowerDNS API 키 설정

```bash
echo "deb [signed-by=/etc/apt/keyrings/auth-49-pub.asc arch=amd64] http://repo.powerdns.com/ubuntu jammy-auth-49 main" | sudo tee /etc/apt/sources.list.d/pdns.list

echo \
"Package: auth*
Pin: origin repo.powerdns.com
Pin-Priority: 600" > /etc/apt/preferences.d/pdns
```

```bash
# wget --quiet -O - https://repo.powerdns.com/FD380FBB-pub.asc | sudo apt-key add -
sudo install -d /etc/apt/keyrings; curl https://repo.powerdns.com/FD380FBB-pub.asc | sudo tee /etc/apt/keyrings/auth-49-pub.asc
```

```bash
systemctl disable --now systemd-resolved

unlink /etc/resolv.conf
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

```bash
sudo apt update
sudo apt-get install -y pdns-server pdns-backend-pgsql
```

```bash
current_pgsql_ver=$(psql -V|awk -F " " '{print $NF}'|awk -F "." '{print $1}')
export PGPASSWORD="$pdns_pwd"
psql -U pdns -h 127.0.0.1 -d pdns < "/usr/share/pdns-backend-pgsql/schema/schema.pgsql.sql"
unset PGPASSWORD

# Copy the example config file
cp -av /usr/share/doc/pdns-backend-pgsql/examples/gpgsql.conf /etc/powerdns/pdns.d/

# Change the values in your PowerDNS Config file to match your saved credentials
sed -i "s/\(^gpgsql-dbname=\).*/\1pdns/" /etc/powerdns/pdns.d/gpgsql.conf
sed -i "s/\(^gpgsql-host=\).*/\1127.0.0.1/" /etc/powerdns/pdns.d/gpgsql.conf
sed -i "s/\(^gpgsql-port=\).*/\15432/" /etc/powerdns/pdns.d/gpgsql.conf
sed -i "s/\(^gpgsql-user=\).*/\1pdnsadmin/" /etc/powerdns/pdns.d/gpgsql.conf
sed -i "s/\(^gpgsql-password=\).*/\1$pdns_pwd/" /etc/powerdns/pdns.d/gpgsql.conf
chmod 640 /etc/powerdns/pdns.d/gpgsql.conf
chown root:pdns /etc/powerdns/pdns.d/gpgsql.conf
```

```bash
sed -i '/\(.*api-key=\)/a api=yes' /etc/powerdns/pdns.conf
sed -i '/\(.*api-key=\).*/a api-key=$pdns_apikey' /etc/powerdns/pdns.conf
```

<details markdown="block">
  <summary>
    코드
  </summary>
  {: .label .label-green }

```bash
sed -i "s/.*webserver=\(yes\|no\)+$/webserver=yes/" "/etc/powerdns/pdns.conf"
```

</details>

```bash
systemctl restart pdns
```

```bash
dig google.com @192.168.0.40
```

<details markdown="block">
  <summary>
    코드
  </summary>
  {: .label .label-green }
  
```bash
; <<>> DiG 9.16.1-Ubuntu <<>> google.com @192.168.0.40
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 53684
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;google.com.                    IN      A

;; Query time: 0 msec
;; SERVER: 192.168.0.40#53(192.168.0.40)
;; WHEN: Wed Oct 25 23:51:48 KST 2023
;; MSG SIZE  rcvd: 39
```

</details>

### PowerDNS-Admin
### Node.js 및 Yarn
### Nginx

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```bash
sudo -u postgres psql -c "\l"
sudo -u postgres psql -d pdns -c "\d"
```

```bash
                                                         List of databases
   Name    |   Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |    Access privileges
-----------+-----------+----------+-----------------+-------------+-------------+------------+-----------+-------------------------
 pdns      | pdnsadmin | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =Tc/pdnsadmin          +
           |           |          |                 |             |             |            |           | pdnsadmin=CTc/pdnsadmin
 postgres  | postgres  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 template0 | postgres  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres            +
           |           |          |                 |             |             |            |           | postgres=CTc/postgres
 template1 | postgres  | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres            +
           |           |          |                 |             |             |            |           | postgres=CTc/postgres
(4 rows)
```

```bash
                   List of relations
 Schema |         Name          |   Type   |   Owner
--------+-----------------------+----------+-----------
 public | comments              | table    | pdnsadmin
 public | comments_id_seq       | sequence | pdnsadmin
 public | cryptokeys            | table    | pdnsadmin
 public | cryptokeys_id_seq     | sequence | pdnsadmin
 public | domainmetadata        | table    | pdnsadmin
 public | domainmetadata_id_seq | sequence | pdnsadmin
 public | domains               | table    | pdnsadmin
 public | domains_id_seq        | sequence | pdnsadmin
 public | records               | table    | pdnsadmin
 public | records_id_seq        | sequence | pdnsadmin
 public | supermasters          | table    | pdnsadmin
 public | tsigkeys              | table    | pdnsadmin
 public | tsigkeys_id_seq       | sequence | pdnsadmin
(13 rows)
```

</details>

```bash
systemctl stop pdns
```

- PowerDNS testing
  
```
pdns_server --daemon=no --guardian=no --loglevel=9
```

- start PostgreSQL Service

```bash
systemctl start pdns
```

```bash
ss -alnp4 | grep pdns
```

```bash
apt-get install nginx \
python3-dev \
libsasl2-dev \
libldap2-dev \
libssl-dev \
libxml2-dev \
libxslt1-dev \
libxmlsec1-dev \
libffi-dev \
pkg-config \
apt-transport-https \
virtualenv \
build-essential \
libmariadb-dev \
git \
libpq-dev \
python3-flask -y
```

- NodeJS

```bash
curl -sL https://deb.nodesource.com/setup_20.x | bash -
apt-get update -y
apt-get install nodejs -y
```

- YARN

```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt update && sudo apt install -y yarn
yarn --version
```

### 저장소 복제

- 선택 1

```bash
git clone https://github.com/PowerDNS-Admin/PowerDNS-Admin.git /var/www/html/pdns
```

- 선택 2

```bash
git clone https://github.com/ngoduykhanh/PowerDNS-Admin.git /var/www/html/pdns
```

- Go into the working directory and enable the virtual environment

```bash
cd "/var/www/html/pdns/"
virtualenv -p python3 flask
source "./flask/bin/activate"
```

- There's a bug with PyYAML 5.4 and Cython 3.0

```bash
sed -i "s/PyYAML==5.4/PyYAML==6.0/g" requirements.txt
```

```bash
pip install --upgrade pip
```

- PostgreSQL 15 이하버전을 사용하는 경우 psycopg2 설치

```bash
pip install psycopg2
```

- PostgreSQL 16 이상버전을 사용하는 경우 psycopg3 설치

```bash
pip install psycopg[binary]
```

- Install the requirements

```bash
pip install -r requirements.txt
```

- Deactivate the virtualenv

```bash
deactivate
```

```bash
# Copy configuration file
cp -av "/var/www/html/pdns/configs/development.py" /var/www/html/pdns/configs/production.py
# Set Filesystem_sessions
sed -i "s/\(^FILESYSTEM_SESSIONS_ENABLED = \).*/\1True/" /var/www/html/pdns/configs/production.py
# Set SALT on the corresponding Parameter
sed -i "s/\(^SALT = \).*/\1\'$pdnsadmin_salt\'/" /var/www/html/pdns/configs/production.py
# Set APIKEY on the corresponding Parameter
sed -i "s/\(^SECRET_KEY = \).*/\1\'$pdns_apikey\'/" /var/www/html/pdns/configs/production.py
# Set MySQL/MariaDB/PGSQL Password on the corresponding Parameter
sed -i "s/\(^SQLA_DB_PASSWORD = \).*/\1\'$pdns_pwd\'/" /var/www/html/pdns/configs/production.py
# Set DB Name
sed -i "s/\(^SQLA_DB_NAME = \).*/\1\'pdns\'/" /var/www/html/pdns/configs/production.py
# Set DB User
sed -i "s/\(^SQLA_DB_USER = \).*/\1\'pdnsadmin\'/" /var/www/html/pdns/configs/production.py
# If you're using PostgreSQL add the following statements to the configuration file
sed -i "s/#import urllib.parse/import urllib.parse/g" /var/www/html/pdns/configs/production.py

# Insert PORT after SQLA_DB_USER
if ! [[ $(grep "SQLA_DB_PORT" /var/www/html/pdns/configs/production.py) ]]; then
    sed -i "/^SQLA_DB_USER.*/a SQLA_DB_PORT = 5432" /var/www/html/pdns/configs/production.py
else
    sed -i "s/\(^SQLA_DB_PORT = \).*/\15432/" /var/www/html/pdns/configs/production.py
fi

# Insert DB URI
cat >> /var/www/html/pdns/configs/production.py <<EOF

### DATABASE - PostgreSQL
## Don't forget to uncomment the import in the top
SQLALCHEMY_DATABASE_URI = 'postgres://{}:{}@{}/{}'.format(
    urllib.parse.quote_plus(SQLA_DB_USER),
    urllib.parse.quote_plus(SQLA_DB_PASSWORD),
    SQLA_DB_HOST,
    SQLA_DB_NAME
)
EOF

# Comment default SQLite DATABASE URI
sed -i "s/\(^SQLALCHEMY_DATABASE_URI = 'sqlite.*\)/#\1/" /var/www/html/pdns/configs/production.py
```

### Front-End/관리 UI 컴파일

```bash
apt-get install -y cmdtest

cd "/var/www/html/pdns/"
source "./flask/bin/activate"

export FLASK_APP=powerdnsadmin/__init__.py
export FLASK_CONF=../configs/production.py
flask db upgrade
npm install -g yarn --force
yarn install --pure-lockfile
flask assets build
deactivate
```

#### Front-End 서비스 설정

- /etc/systemd/system/pdnsadmin.service

```bash
tee /etc/systemd/system/pdnsadmin.service << EOF
[Unit]
Description=PowerDNS-Admin
Requires=pdnsadmin.socket
After=network.target

[Service]
PIDFile=/run/pdnsadmin/pid
User=pdns
Group=pdns
Environment="FLASK_CONF=../configs/production.py"
WorkingDirectory=/var/www/html/pdns
ExecStart=/var/www/html/pdns/flask/bin/gunicorn --pid /run/pdnsadmin/pid --bind unix:/run/pdnsadmin/socket 'powerdnsadmin:create_app()'
ExecReload=/bin/kill -s HUP \$MAINPID
ExecStop=/bin/kill -s TERM \$MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

- /etc/systemd/system/pdnsadmin.socket

```bash
tee /etc/systemd/system/pdnsadmin.socket << EOF
[Unit]
Description=PowerDNS-Admin socket

[Socket]
ListenStream=/run/pdnsadmin/socket

[Install]
WantedBy=sockets.target
EOF
```

- /etc/nginx/sites-enabled/powerdns-admin.conf

<details markdown="block">
  <summary>
    코드
  </summary>
  {: .label .label-green }

```bash
tee /etc/nginx/sites-enabled/powerdns-admin.conf << EOF
server {
  listen  *:80;
  server_name               pdnsadmin.example.com;

  index                     index.html index.htm index.php;
  root                      /var/www/html/pdns;
  access_log                /var/log/nginx/pdnsadmin_access.log combined;
  error_log                 /var/log/nginx/pdnsadmin_error.log;

  client_max_body_size              10m;
  client_body_buffer_size           128k;
  proxy_redirect                    off;
  proxy_connect_timeout             90;
  proxy_send_timeout                90;
  proxy_read_timeout                90;
  proxy_buffers                     32 4k;
  proxy_buffer_size                 8k;
  proxy_set_header                  Host \$host;
  proxy_set_header                  X-Real-IP \$remote_addr;
  proxy_set_header                  X-Forwarded-For \$proxy_add_x_forwarded_for;
  proxy_headers_hash_bucket_size    64;

  location ~ ^/static/  {
    include  /etc/nginx/mime.types;
    root /var/www/html/pdns/powerdnsadmin;

    location ~*  \.(jpg|jpeg|png|gif)$ {
    expires 365d;
    }

    location ~* ^.+.(css|js)$ {
    expires 7d;
    }
  }

  location / {
    proxy_pass            http://unix:/run/pdnsadmin/socket;
    proxy_read_timeout    120;
    proxy_connect_timeout 120;
    proxy_redirect        off;
  }
}
EOF
```

</details>


```bash
chown -R pdns:www-data "/var/www/html/pdns"
nginx -t && systemctl restart nginx
```

```bash
echo "d /run/pdnsadmin 0755 pdns pdns -" >> "/etc/tmpfiles.d/pdnsadmin.conf"
mkdir -p "/run/pdnsadmin/"
chown -R pdns: "/run/pdnsadmin/"
chown -R pdns: "/var/www/html/pdns/powerdnsadmin/"
```

```bash
systemctl daemon-reload && systemctl enable --now pdnsadmin.service pdnsadmin.socket
```

## 참조
- [PowerDNS 공식 리포지토리](http://repo.powerdns.com/#ubuntu)
- [PowerDNS 설치 가이](https://docs.brconsulting.info/en/docs/network/powerdns/02-pdns-installation/)
- [PostgreSQL 공식 문서](https://www.postgresql.org/download/linux/ubuntu/)
- [NodeJS 공식 문서](https://deb.nodesource.com/node_20.x/pool/main/n/nodejs)

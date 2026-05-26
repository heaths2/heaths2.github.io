---
title: RSyslog 설정법
author: G.G
date: 2025-05-21 12:52 +0900
categories: [Blog, Provisioning ]
tags: [Provisioning , syslog, logrotate]
---

## 📘 개요 (Overview)
이 문서는 rsyslog를 활용하여 FortiGate 및 SECUI 보안 장비에서 전송된 로그를 IP 주소 및 로그 심각도(Severity) 기준으로 분리 저장하는 방법을 설명합니다.
각 장비에서 전송하는 로그 메시지를 장비별로 분류하여, 지정한 디렉토리에 자동으로 저장되도록 구성하였으며, 파일명은 날짜에 따라 자동 생성됩니다.
이 설정을 통해 다음과 같은 목적을 달성할 수 있습니다.
- 장비별 로그를 별도 디렉토리에 저장하여 운영 가시성 확보
- 로그 파일의 자동 생성 및 폴더 분리로 장비별 트래픽 추적 용이
- 추후 Kibana 등 로그 분석 도구 연동 시 수집 효율성 증대

## 🧱 아키텍처 (Architecture)


| 구분     | 장비명         | IP 주소       | 로그 심각도 | 로그 Facility |
| ------ | ----------- | ----------- | ------ | ----------- |
| 내부 방화벽 | FortiGate   | 10.1.50.1   | notice | local7      |
| 내부 IPS | SECUI INT#1 | 10.1.50.252 | info   | local7      |
| 내부 IPS | SECUI INT#2 | 10.1.50.253 | info   | local7      |
| 외부 IPS | SECUI EXT#1 | 10.1.50.17  | info   | local7      |
| 외부 IPS | SECUI EXT#2 | 10.1.50.18  | info   | local7      |


```bash
[보안 장비 로그 전송]
       ↓
  [Rsyslog 수신 서버]
       ↓
[필터 조건에 따라 분기 처리]
    ├─ FortiGate → /data/log/10.1.50.1/syslog
    ├─ SECUI INT#1 → /data/log/SECUI-INT#1/syslog
    ├─ SECUI INT#2 → /data/log/SECUI-INT#2/syslog
    ├─ SECUI EXT#1 → /data/log/SECUI-EXT#1/syslog
    └─ SECUI EXT#2 → /data/log/SECUI-EXT#2/syslog
```

## ⚙️ 사용법

### RSyslog 

- rsyslog.conf 설정

```yaml
# rsyslog configuration file

# For more information see /usr/share/doc/rsyslog-*/rsyslog_conf.html
# or latest version online at http://www.rsyslog.com/doc/rsyslog_conf.html 
# If you experience problems, see http://www.rsyslog.com/doc/troubleshoot.html

#### GLOBAL DIRECTIVES ####

# Where to place auxiliary files
global(workDirectory="/var/lib/rsyslog")

# Use default timestamp format
module(load="builtin:omfile" Template="RSYSLOG_TraditionalFileFormat")

#### MODULES ####

module(load="imuxsock"    # provides support for local system logging (e.g. via logger command)
       SysSock.Use="off") # Turn off message reception via local log socket; 
     # local messages are retrieved through imjournal now.
module(load="imjournal"      # provides access to the systemd journal
       UsePid="system" # PID nummber is retrieved as the ID of the process the journal entry originates from
       FileCreateMode="0644" # Set the access permissions for the state file
       StateFile="imjournal.state") # File to store the position in the journal
#module(load="imklog") # reads kernel messages (the same are read from journald)
#module(load="immark") # provides --MARK-- message capability

# Include all config files in /etc/rsyslog.d/
include(file="/etc/rsyslog.d/*.conf" mode="optional")

# Provides UDP syslog reception
# for parameters see http://www.rsyslog.com/doc/imudp.html
module(load="imudp") # needs to be done just once
input(type="imudp" port="514")

# Provides TCP syslog reception
# for parameters see http://www.rsyslog.com/doc/imtcp.html
module(load="imtcp") # needs to be done just once
input(type="imtcp" port="514")

#### RULES ####

# Log all kernel messages to the console.
# Logging much else clutters up the screen.
kern.*                                                 /dev/console

# Log anything (except mail) of level info or higher.
# Don't log private authentication messages!
#*.info;mail.none;authpriv.none;cron.none                /var/log/messages
*.info;mail.none;authpriv.none;cron.none;local7.none    /var/log/messages

# The authpriv file has restricted access.
authpriv.*                                              /var/log/secure

# Log all the mail messages in one place.
mail.*                                                  -/var/log/maillog


# Log cron stuff
cron.*                                                  /var/log/cron

# Everybody gets emergency messages
*.emerg                                                 :omusrmsg:*

# Save news errors of level crit and higher in a special file.
uucp,news.crit                                          /var/log/spooler

# Save boot messages also to boot.log
#local7.*                                                /var/log/boot.log

# ### sample forwarding rule ###
#action(type="omfwd"  
# # An on-disk queue is created for this action. If the remote host is
# # down, messages are spooled to disk and sent when it is up again.
#queue.filename="fwdRule1"       # unique name prefix for spool files
#queue.maxdiskspace="1g"         # 1gb space limit (use as much as possible)
#queue.saveonshutdown="on"       # save messages to disk on shutdown
#queue.type="LinkedList"         # run asynchronously
#action.resumeRetryCount="-1"    # infinite retries if host is down
# # Remote Logging (we use TCP for reliable delivery)
# # remote_host is: name/ip, e.g. 192.168.0.1, port optional e.g. 10514
#Target="remote_host" Port="XXX" Protocol="tcp")
```
{: file='/etc/rsyslog.conf'}

- rsyslog.d/default.conf 생성 및 설정

```yaml
#### ✅ 템플릿 정의 ####
# 파일 포맷
#template (name="FileFormat" type="string" string="/data/log/%HOSTNAME%/syslog-%$year%%$month%%$day%")
template(name="FileFormat" type="list") {
    constant(value="/data/log/")
    property(name="hostname")
    constant(value="/syslog")
}

# 메시지 포맷
template(name="DateTime" type="list") {
    constant(value="[")
    property(name="timereported" dateformat="year")
    constant(value="-")
    property(name="timereported" dateformat="month")
    constant(value="-")
    property(name="timereported" dateformat="day")
    constant(value=" ")
    property(name="timereported" dateformat="hour")
    constant(value=":")
    property(name="timereported" dateformat="minute")
    constant(value=":")
    property(name="timereported" dateformat="second")
    constant(value="] ")
    property(name="hostname")
    constant(value=" >> ")
    property(name="syslogtag")
#    property(name="msg" spifno1stsp="on" droplastlf="on")
    property(name="rawmsg")
    constant(value="\n")
}

#### ✅ FortiGate 처리: local7.notice + IP별 분기 저장 ####

# FortiGate 로그 중 local7.notice만 필터링 + IP별 태깅
if ($fromhost-ip == '10.1.50.1' and $syslogfacility-text == 'local7' and $syslogseverity-text == 'notice') then {
    set $!ip_tag = "10.1.50.1";
    action(
        type="omfile"
        dynaFile="FileFormat"
        template="DateTime"
        createDirs="on"
    )
}

#### ✅ SECUI 처리: local7.info + IP별 분기 저장 ####

if ($fromhost-ip == '10.1.50.252' and $syslogfacility-text == 'local7' and $syslogseverity-text == 'info') then {
    set $!ip_tag = "10.1.50.252";
    action(
        type="omfile"
        file="/data/log/SECUI-INT#1/syslog"
        template="DateTime"
        createDirs="on"
    )
    stop
}
if ($fromhost-ip == '10.1.50.253' and $syslogfacility-text == 'local7' and $syslogseverity-text == 'info') then {
    set $!ip_tag = "10.1.50.253";
    action(
        type="omfile"
        file="/data/log/SECUI-INT#2/syslog"
        template="DateTime"
        createDirs="on"
    )
    stop
}
if ($fromhost-ip == '10.1.50.17' and $syslogfacility-text == 'local7' and $syslogseverity-text == 'info') then {
    set $!ip_tag = "10.1.50.17";
    action(
        type="omfile"
        file="/data/log/SECUI-EXT#1/syslog"
        template="DateTime"
        createDirs="on"
    )
    stop
}
if ($fromhost-ip == '10.1.50.18' and $syslogfacility-text == 'local7' and $syslogseverity-text == 'info') then {
    set $!ip_tag = "10.1.50.18";
    action(
        type="omfile"
        file="/data/log/SECUI-EXT#2/syslog"
        template="DateTime"
        createDirs="on"
    )
    stop
}
```
{: file='/etc/rsyslog.d/default.conf'}

### Logrotate 

- RHEL 계열

```bash
# /data/log/*/syslog - 해당 경로의 로그 파일에 대한 logrotate 설정
# 매일 실행
# 만약 로그 파일이 없어도 계속 진행 (에러 무시)
# 최대 180개의 로그 파일 유지
# 최대 180일 이상된 로그 파일 삭제
# 로그 파일을 압축 (rotate 이후에 압축 진행)
# 압축된 로그 파일을 지연 압축 (다음 실행 시 압축 진행)
# 현재 로그파일 복사 후 원본 로그파일 크기 0으로 생성
# 비어있는 로그 파일은 회전하지 않음
# 로그 파일 생성 시 권한 설정
# 로그 파일에 날짜 확장자 추가
# 날짜 형식 지정 (%Y: 연도, %m: 월, %d: 일)
# logrotate 실행 시 sharedscripts 이후에 postrotate 스크립트 실행
# postrotate 스크립트 - rsyslog-rotate 실행
/data/log/*/syslog {
    daily
    missingok
    rotate 180
    maxage 180
    compress
    compresscmd /usr/bin/zstd
    uncompresscmd /usr/bin/unzstd
    compressoptions -19 -T0
    compressext .zst
    dateext
    dateformat -%Y-%m-%d
    delaycompress
    missingok
    notifempty
    sharedscripts

    postrotate
        /usr/bin/systemctl -s HUP kill rsyslog.service >/dev/null 2>&1 || true
    endscript
}
```
{: file='/etc/logrotate.d/firewall'}

- Debian/Ubuntu 계열

```bash
# /data/log/*/syslog - 해당 경로의 로그 파일에 대한 logrotate 설정
# 매일 실행
# 만약 로그 파일이 없어도 계속 진행 (에러 무시)
# 최대 180개의 로그 파일 유지
# 최대 180일 이상된 로그 파일 삭제
# 로그 파일을 압축 (rotate 이후에 압축 진행)
# 압축된 로그 파일을 지연 압축 (다음 실행 시 압축 진행)
# 현재 로그파일 복사 후 원본 로그파일 크기 0으로 생성
# 비어있는 로그 파일은 회전하지 않음
# 로그 파일 생성 시 권한 설정
# 로그 파일에 날짜 확장자 추가
# 날짜 형식 지정 (%Y: 연도, %m: 월, %d: 일)
# logrotate 실행 시 sharedscripts 이후에 postrotate 스크립트 실행
# postrotate 스크립트 - rsyslog-rotate 실행
/data/log/*/syslog {
    daily
    missingok
    rotate 180
    maxage 180
    compress
    compresscmd /usr/bin/zstd
    uncompresscmd /usr/bin/unzstd
    compressoptions -19 -T0
    compressext .zst
    dateext
    dateformat -%Y-%m-%d
    delaycompress
    missingok
    notifempty
    sharedscripts
    create 0640 syslog adm

    postrotate
        invoke-rc.d rsyslog rotate >/dev/null
    endscript
}
```
{: file='/etc/logrotate.d/firewall'}

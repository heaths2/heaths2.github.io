---
title: ELK Stack 구성
author: G.G
date: 2025-01-10 16:01:00 +0900
categories: [Blog, Orchestration]
tags: [ansible, awx, postgresql]
---

> 관련글 :
> [ Kubespray 설치방법 (1)](https://heaths2.github.io/posts/kubespray_install/)
> [ Kubespray 설치방법 (1)](https://heaths2.github.io/posts/kubespray_install/)

## 개요
ELK Stack(Elasticsearch, Logstash, Kibana)은 로그 데이터를 수집, 저장, 분석, 시각화하는 데 최적화된 오픈 소스 소프트웨어 스택입니다. 본 매뉴얼은 여러 대의 FortiGate 방화벽에서 생성되는 로그를 중앙 집중식으로 관리하고 시각화하기 위한 ELK Stack 시스템의 구성과 설정 방법을 다룹니다.

## 목적
1. FortiGate 방화벽에서 생성되는 로그 데이터를 한 곳에 모아 관리.
2. 실시간 로그 분석 및 검색 기능 제공.
3. 사용자 정의 대시보드를 통한 직관적인 데이터 시각화.
4. 네트워크 보안 이벤트의 신속한 모니터링 및 대응 체계 강화.

## 특징
1. 데이터 수집 및 변환 (Logstash):
- 다양한 입력 플러그인을 사용해 FortiGate 장비에서 데이터를 수집.
- 데이터 필터링 및 변환 기능을 통해 로그 형식을 표준화.
- 필요에 따라 GeoIP, 날짜 변환 등의 추가 처리가 가능.

2. 데이터 저장 및 검색 (Elasticsearch):
- 수집된 데이터를 구조화된 방식으로 저장.
- 빠른 검색과 고성능 쿼리 실행 가능.
- 스냅샷 및 클러스터 구성을 통한 고가용성과 확장성 지원.

3. 데이터 시각화 (Kibana):
- 사용자가 직접 정의할 수 있는 대시보드 제공.
- 실시간으로 네트워크 트래픽, 보안 이벤트 등의 데이터를 시각화.
- 다양한 그래프와 차트를 사용해 데이터를 효과적으로 분석.

4. 확장성 및 유연성:
- 수집 장비(FortiGate)의 개수 증가에 따라 손쉬운 확장 가능.
- 다양한 데이터 소스와의 통합 지원.

5. 중앙 집중화:
- 여러 대의 FortiGate 장비에서 발생하는 로그를 한 곳으로 통합.
- 중앙에서 관리함으로써 로그 분실 방지 및 분석 효율성 증가.

6. 보안 및 접근 제어:
- Elasticsearch와 Kibana에 대한 역할 기반 액세스 제어(RBAC) 적용.
- SSL/TLS를 통한 데이터 전송 암호화.

## 환경 구성

### JAVA JVM 설치

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
java -version
echo "
# Open JDK 17
JAVA_HOME=\"/usr/lib/jvm/java-17-openjdk-amd64/\"\
" >> /etc/environment
source /etc/environment
echo $JAVA_HOME
```

### Elasticsearch, Logstash, Kibana 설치

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
sudo apt-get install apt-transport-https
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
sudo apt-get update && sudo apt-get install -y elasticsearch logstash kibana
```

## Elasticsearch, Logstash, Kibana 설치 및 설정 가이드

### Elasticsearch 구성 설정
1. Elasticsearch 환경 변수 설정

```bash
echo "
# Elasticsearch
ES_HOME=/usr/share/elasticsearch
ES_PATH_CONF=/etc/elasticsearch" \
>> /etc/environment
source /etc/environment
echo $ES_HOME
echo $ES_PATH_CONF
```

2. Elasticsearch 설정 파일 수정

```bash
sed -i -e "s/#network.host:.*/network.host: 0.0.0.0/" \
       -e 's/#http.port: 9200/http.port: 9200/' \
       -e 's/path.data:.*/path.data: \/data\/elasticsearch/' \
       -e '/#bootstrap.memory_lock: true/ s/^#//' \
       -e "s/#cluster.name:.*/cluster.name: $(hostname -s)/" \
       -e "s/#node.name:.*/node.name: $(hostname -s)/" \
       -e '92s/enabled: true/enabled: false/' \
       -e '94s/enabled: true/enabled: false/' \
       -e '98s/enabled: true/enabled: false/' \
       -e '103s/enabled: true/enabled: false/' \
       -e '/#transport.host: 0.0.0.0/a\transport.port: 9300' /etc/elasticsearch/elasticsearch.yml
```

3. Elasticsearch 서비스 설정 수정

```bash
sed -i '/\[Install\]/i # LimitMEMLOCK\nLimitMEMLOCK=infinity\n' /usr/lib/systemd/system/elasticsearch.service
```

4. Elasticsearch JVM 메모리 설정

```bash
sed -i -e 's/## -Xms4g/-Xms16g/' \
       -e 's/## -Xmx4g/-Xmx16g/' /etc/elasticsearch/jvm.options
```

5. Elasticsearch 서비스 시작 및 상태 확인

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch.service
curl -X GET http://localhost:9200/_cat/health?v
curl -X GET http://localhost:9200/_cluster/health?pretty
curl http://localhost:9200
```

```bash
{
  "name" : "nt-els01",
  "cluster_name" : "nt-els01",
  "cluster_uuid" : "Dqa7sGJIS7ivn96BiByPJQ",
  "version" : {
    "number" : "8.13.4",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "da95df118650b55a500dcc181889ac35c6d8da7c",
    "build_date" : "2024-05-06T22:04:45.107454559Z",
    "build_snapshot" : false,
    "lucene_version" : "9.10.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

> - Elasticsearch 클러스터 정보 확인 출력 예시
{: .prompt-tip }

### Kibana 구성 설정
1. Kibana 환경 변수 설정

```bash
echo "
# Elasticsearch
KIBANA_HOME=/usr/share/kibana
KIBANA_PATH_CONFIG=/etc/kibana" \
>> /etc/environment
source /etc/environment
echo $KIBANA_HOME
echo $KIBANA_PATH_CONFIG
```

2. Kibana 설정 파일 수정

```bash
sed -i -e '/#server.port: 5601/ s/^#//' \
       -e 's/#server.host:.*/server.host: "0.0.0.0"/' \
       -e "s/#server.name:.*/server.name: \"$(hostname -s)\"/" \
       -e 's/#i18n.locale: "en"/i18n.locale: "ko_KR"/' \
       -e '/#elasticsearch.hosts:/ s/^#//' /etc/kibana/kibana.yml
```

3. Kibana 서비스 시작 및 상태 확인

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kibana.service
```

### Logstash FortiGate 로그 파싱 및 Elasticsearch 연동 설정
1. Logstash 설정 파일 생성

```bash
cat <<EOF > /etc/logstash/conf.d/fortigate.conf
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => {
      "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] %{DATA:device} >> (?:date=%{YEAR:year}-%{MONTHNUM:month}-%{MONTHDAY:day} )?(?:time=%{HOUR:hour}:%{MINUTE:minute}:%{SECOND:second} )?(?:devname=\"%{DATA:devname}\" )?(?:devid=\"%{DATA:devid}\" )?(?:eventtime=%{NUMBER:eventtime} )?(?:tz=\"%{DATA:tz}\" )?(?:logid=\"%{DATA:logid}\" )?(?:type=\"%{DATA:type}\" )?(?:subtype=\"%{DATA:subtype}\" )?(?:level=\"%{DATA:level}\" )?(?:vd=\"%{DATA:vd}\" )?(?:srcip=%{IP:srcip} )?(?:srcport=%{NUMBER:srcport} )?(?:srcintf=\"%{DATA:srcintf}\" )?(?:srcintfrole=\"%{DATA:srcintfrole}\" )?(?:dstip=%{IP:dstip} )?(?:dstport=%{NUMBER:dstport} )?(?:dstintf=\"%{DATA:dstintf}\" )?(?:dstintfrole=\"%{DATA:dstintfrole}\" )?(?:srccountry=\"%{DATA:srccountry}\" )?(?:dstcountry=\"%{DATA:dstcountry}\" )?(?:sessionid=%{NUMBER:sessionid} )?(?:proto=%{NUMBER:proto} )?(?:action=\"%{DATA:action}\" )?(?:policyid=%{NUMBER:policyid} )?(?:policytype=\"%{DATA:policytype}\" )?(?:poluuid=\"%{UUID:poluuid}\" )?(?:service=\"%{DATA:service}\" )?(?:trandisp=\"%{DATA:trandisp}\" )?(?:transip=%{IP:transip} )?(?:transport=%{NUMBER:transport} )?(?:duration=%{NUMBER:duration} )?(?:sentbyte=%{NUMBER:sentbyte} )?(?:rcvdbyte=%{NUMBER:rcvdbyte} )?(?:sentpkt=%{NUMBER:sentpkt} )?(?:rcvdpkt=%{NUMBER:rcvdpkt} )?(?:appcat=\"%{DATA:appcat}\" )?(?:mastersrcmac=\"%{MAC:mastersrcmac}\" )?(?:srcmac=\"%{MAC:srcmac}\" )?(?:srcserver=%{NUMBER:srcserver})?"
    }
  }

  # prune 필터를 사용하여 불필요한 필드 제거
  prune {
    whitelist_names => [ "timestamp", "device", "devname", "devid", "eventtime", "tz", "logid", "type", "subtype", "level", "vd", "srcip", "srcport", "srcintf", "srcintfrole", "dstip", "dstport", "dstintf", "dstintfrole", "srccountry", "dstcountry", "sessionid", "proto", "action", "policyid", "policytype", "poluuid", "service", "trandisp", "transip", "transport", "duration", "sentbyte", "rcvdbyte", "sentpkt", "rcvdpkt", "appcat", "mastersrcmac", "srcmac", "srcserver" ]
  }

  # mutate 필터를 사용하여 특정 필드 제거
  mutate {
    remove_field => [ "event.original", "message" ]  # 필요 없는 필드 제거
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "fortigate-logs-%{+YYYY.MM.dd}"
  }
#  stdout { codec => rubydebug }
}
EOF
```
{: file='fortigate.conf'}

2. Logstash 설정 테스트 및 자동 재로드

```bash
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/fortigate.conf --config.test_and_exit
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/fortigate.conf --config.reload.automatic
```

## Filebeat 설치 및 구성 TOC
FortiGate 로그 수집서버에서 진행.

1. Filebeat 설치

```bash
echo "deb https://artifacts.elastic.co/packages/oss-8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo apt-get update && sudo apt-get install filebeat
```

2. Filebeat 입력 활성화

```bash
sed -i -e 's/enabled: false/enabled: true/' \
           /etc/filebeat/filebeat.yml
```

3. `/etc/filebeat/filebeat.yml`{: .filepath} Filebeat 구성 파일 생성

```yaml
###################### Filebeat Configuration Example #########################

# This file is an example configuration file highlighting only the most common
# options. The filebeat.reference.yml file from the same directory contains all the
# supported options with more comments. You can use it as a reference.
#
# You can find the full configuration reference here:
# https://www.elastic.co/guide/en/beats/filebeat/index.html

# For more available modules and options, please see the filebeat.reference.yml sample
# configuration file.

# ============================== Filebeat inputs ===============================

filebeat.inputs:

# Each - is an input. Most options can be set at the input level, so
# you can use different inputs for various configurations.
# Below are the input-specific configurations.

# filestream is an input for collecting log messages from files.
- type: filestream

  # Unique ID among all inputs, an ID is required.
  #  id: my-filestream-id

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    #- /var/log/*.log
    #- c:\programdata\elasticsearch\logs\*
    - /data/log/fg-100e/syslog
    - /data/log/fg-100f/syslog
    - /data/log/fg-200e/syslog
    - /data/log/fg-300d/syslog
    - /data/log/fg-400f/syslog
    - /data/log/fg-60d/syslog
    - /data/log/fg-60e/syslog

  fields:
    service: fortigate


  # Exclude lines. A list of regular expressions to match. It drops the lines that are
  # matching any regular expression from the list.
  # Line filtering happens after the parsers pipeline. If you would like to filter lines
  # before parsers, use include_message parser.
  #exclude_lines: ['^DBG']

  # Include lines. A list of regular expressions to match. It exports the lines that are
  # matching any regular expression from the list.
  # Line filtering happens after the parsers pipeline. If you would like to filter lines
  # before parsers, use include_message parser.
  #include_lines: ['^ERR', '^WARN']

  # Exclude files. A list of regular expressions to match. Filebeat drops the files that
  # are matching any regular expression from the list. By default, no files are dropped.
  #prospector.scanner.exclude_files: ['.gz$']

  # Optional additional fields. These fields can be freely picked
  # to add additional information to the crawled log files for filtering
  #fields:
  #  level: debug
  #  review: 1

# ============================== Filebeat modules ==============================

filebeat.config.modules:
  # Glob pattern for configuration loading
  path: ${path.config}/modules.d/*.yml

  # Set to true to enable config reloading
  reload.enabled: false

  # Period on which files under path should be checked for changes
  #reload.period: 10s

# ======================= Elasticsearch template setting =======================

setup.template.settings:
  index.number_of_shards: 1
  #index.codec: best_compression
  #_source.enabled: false


# ================================== General ===================================

# The name of the shipper that publishes the network data. It can be used to group
# all the transactions sent by a single shipper in the web interface.
#name:

# The tags of the shipper are included in their field with each
# transaction published.
#tags: ["service-X", "web-tier"]

# Optional fields that you can specify to add additional information to the
# output.
#fields:
#  env: staging

# ================================= Dashboards =================================
# These settings control loading the sample dashboards to the Kibana index. Loading
# the dashboards is disabled by default and can be enabled either by setting the
# options here or by using the `setup` command.
#setup.dashboards.enabled: false

# The URL from where to download the dashboard archive. By default, this URL
# has a value that is computed based on the Beat name and version. For released
# versions, this URL points to the dashboard archive on the artifacts.elastic.co
# website.
#setup.dashboards.url:

# =================================== Kibana ===================================

# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  #host: "localhost:5601"

  # Kibana Space ID
  # ID of the Kibana Space into which the dashboards should be loaded. By default,
  # the Default Space will be used.
  #space.id:

# =============================== Elastic Cloud ================================

# These settings simplify using Filebeat with the Elastic Cloud (https://cloud.elastic.co/).

# The cloud.id setting overwrites the `output.elasticsearch.hosts` and
# `setup.kibana.host` options.
# You can find the `cloud.id` in the Elastic Cloud web UI.
#cloud.id:

# The cloud.auth setting overwrites the `output.elasticsearch.username` and
# `output.elasticsearch.password` settings. The format is `<user>:<pass>`.
#cloud.auth:

# ================================== Outputs ===================================

# Configure what output to use when sending the data collected by the beat.

# ---------------------------- Elasticsearch Output ----------------------------
#output.elasticsearch:
  # Array of hosts to connect to.
#  hosts: ["localhost:9200"]

  # Performance preset - one of "balanced", "throughput", "scale",
  # "latency", or "custom".
#  preset: balanced

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  #username: "elastic"
  #password: "changeme"

# ------------------------------ Logstash Output -------------------------------
output.logstash:
  # The Logstash hosts
  hosts: ["10.1.10.11:5044"]

  # Optional SSL. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

# ================================= Processors =================================
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

    #output.console.pretty: true

# ================================== Logging ===================================

# Sets log level. The default log level is info.
# Available log levels are: error, warning, info, debug
#logging.level: debug

# At debug level, you can selectively enable logging only for some components.
# To enable all selectors, use ["*"]. Examples of other selectors are "beat",
# "publisher", "service".
#logging.selectors: ["*"]

# ============================= X-Pack Monitoring ==============================
# Filebeat can export internal metrics to a central Elasticsearch monitoring
# cluster.  This requires xpack monitoring to be enabled in Elasticsearch.  The
# reporting is disabled by default.

# Set to true to enable the monitoring reporter.
#monitoring.enabled: false

# Sets the UUID of the Elasticsearch cluster under which monitoring data for this
# Filebeat instance will appear in the Stack Monitoring UI. If output.elasticsearch
# is enabled, the UUID is derived from the Elasticsearch cluster referenced by output.elasticsearch.
#monitoring.cluster_uuid:

# Uncomment to send the metrics to Elasticsearch. Most settings from the
# Elasticsearch outputs are accepted here as well.
# Note that the settings should point to your Elasticsearch *monitoring* cluster.
# Any setting that is not set is automatically inherited from the Elasticsearch
# output configuration, so if you have the Elasticsearch output configured such
# that it is pointing to your Elasticsearch monitoring cluster, you can simply
# uncomment the following line.
#monitoring.elasticsearch:

# ============================== Instrumentation ===============================

# Instrumentation support for the filebeat.
#instrumentation:
    # Set to true to enable instrumentation of filebeat.
    #enabled: false

    # Environment in which filebeat is running on (eg: staging, production, etc.)
    #environment: ""

    # APM Server hosts to report instrumentation results to.
    #hosts:
    #  - http://localhost:8200

    # API Key for the APM Server(s).
    # If api_key is set then secret_token will be ignored.
    #api_key:

    # Secret token for the APM Server(s).
    #secret_token:


# ================================= Migration ==================================

# This allows to enable 6.7 migration aliases
#migration.6_to_7.enabled: true
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat/filebeat.log
  name: filebeat
  keepfiles: 7
  permissions: 0640
```
{: file='filebeat.yml'}

4. Filebeat 테스트

```bash
filebeat test config
```

5. Filebeat 실행

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

## 튜닝
튜닝된 매핑 설정은 로그 데이터를 효율적으로 처리하고, 검색 및 집계 성능을 향상시키는 데 중점을 둡니다. 아래는 FortiGate 로그 매핑을 튜닝하는 방법과 출력 예시입니다.

### 튜닝 목표
1. 필드 자료형 최적화: text, keyword, ip, long, date 등을 적절히 사용.
2. 불필요한 필드 제외: 로그에서 자주 사용되지 않는 필드는 제거.
3. 동적 매핑 제한: 동적 필드 생성 방지로 불필요한 리소스 사용 방지.

### 매핑 튜닝 설정

```bash
PUT /fortigate-logs-template
{
  "index_patterns": ["fortigate-logs-*"],
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "action": { "type": "keyword" },
      "appcat": { "type": "keyword" },
      "device": { "type": "keyword" },
      "devid": { "type": "keyword" },
      "devname": { "type": "keyword" },
      "dstcountry": { "type": "keyword" },
      "dstintf": { "type": "keyword" },
      "dstintfrole": { "type": "keyword" },
      "dstip": { "type": "ip" },
      "dstport": { "type": "integer" },
      "duration": { "type": "long" },
      "eventtime": { "type": "date" },
      "level": { "type": "keyword" },
      "logid": { "type": "keyword" },
      "mastersrcmac": { "type": "keyword" },
      "policyid": { "type": "long" },
      "policytype": { "type": "keyword" },
      "poluuid": { "type": "keyword" },
      "proto": { "type": "keyword" },
      "rcvdbyte": { "type": "long" },
      "rcvdpkt": { "type": "long" },
      "sentbyte": { "type": "long" },
      "sentpkt": { "type": "long" },
      "service": { "type": "keyword" },
      "sessionid": { "type": "keyword" },
      "srccountry": { "type": "keyword" },
      "srcintf": { "type": "keyword" },
      "srcintfrole": { "type": "keyword" },
      "srcip": { "type": "ip" },
      "srcmac": { "type": "keyword" },
      "srcport": { "type": "integer" },
      "subtype": { "type": "keyword" },
      "trandisp": { "type": "keyword" },
      "transip": { "type": "ip" },
      "transport": { "type": "integer" },
      "type": { "type": "keyword" },
      "tz": { "type": "keyword" },
      "vd": { "type": "keyword" }
    }
  }
}
```

3. 튜닝된 매핑 출력 예시

```bash
{
  "fortigate-logs-2025.01.10" : {
    "mappings" : {
      "properties" : {
        "@timestamp" : {
          "type" : "date"
        },
        "action" : {
          "type" : "keyword"
        },
        "appcat" : {
          "type" : "keyword"
        },
        "device" : {
          "type" : "keyword"
        },
        "devid" : {
          "type" : "keyword"
        },
        "devname" : {
          "type" : "text"
        },
        "dstcountry" : {
          "type" : "text"
        },
        "dstintf" : {
          "type" : "text"
        },
        "dstintfrole" : {
          "type" : "keyword"
        },
        "dstip" : {
          "type" : "text"
        },
        "dstport" : {
          "type" : "long"
        },
        "duration" : {
          "type" : "keyword"
        },
        "eventtime" : {
          "type" : "keyword"
        },
        "level" : {
          "type" : "keyword"
        },
        "logid" : {
          "type" : "keyword"
        },
        "mastersrcmac" : {
          "type" : "keyword"
        },
        "policyid" : {
          "type" : "long"
        },
        "policytype" : {
          "type" : "keyword"
        },
        "poluuid" : {
          "type" : "keyword"
        },
        "proto" : {
          "type" : "keyword"
        },
        "rcvdbyte" : {
          "type" : "long"
        },
        "rcvdpkt" : {
          "type" : "keyword"
        },
        "sentbyte" : {
          "type" : "long"
        },
        "sentpkt" : {
          "type" : "keyword"
        },
        "service" : {
          "type" : "text"
        },
        "sessionid" : {
          "type" : "keyword"
        },
        "srccountry" : {
          "type" : "text"
        },
        "srcintf" : {
          "type" : "text"
        },
        "srcintfrole" : {
          "type" : "keyword"
        },
        "srcip" : {
          "type" : "text"
        },
        "srcmac" : {
          "type" : "keyword"
        },
        "srcport" : {
          "type" : "long"
        },
        "srcserver" : {
          "type" : "keyword"
        },
        "subtype" : {
          "type" : "keyword"
        },
        "timestamp" : {
          "type" : "text"
        },
        "trandisp" : {
          "type" : "keyword"
        },
        "transip" : {
          "type" : "text"
        },
        "transport" : {
          "type" : "long"
        },
        "type" : {
          "type" : "keyword"
        },
        "tz" : {
          "type" : "text"
        },
        "vd" : {
          "type" : "text"
        }
      }
    }
  }
}
```

> - 튜닝된 매핑 출력 예시
{: .prompt-tip }

### 주요 변경 사항
1. 필드 자료형 최적화:
- srcip, dstip, transip → ip 자료형.
- dstport, srcport, transport → integer.
- rcvdbyte, sentbyte, duration → long.
- devname, dstcountry → keyword (텍스트 분석이 필요하지 않음).
2. 사용하지 않는 필드 제거:
- 필요하지 않은 필드는 매핑에서 제거.
3. 일관성 유지:
- 동일한 데이터 유형은 동일한 자료형으로 통합.


### 템플릿 적용
1. 템플릿 적용

```bash
curl -X PUT "http://localhost:9200/_template/fortigate-logs-template" -H 'Content-Type: application/json' -d '
PUT /fortigate-logs-template
{
  "index_patterns": ["fortigate-logs-*"],
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "action": { "type": "keyword" },
      "appcat": { "type": "keyword" },
      "device": { "type": "keyword" },
      "devid": { "type": "keyword" },
      "devname": { "type": "keyword" },
      "dstcountry": { "type": "keyword" },
      "dstintf": { "type": "keyword" },
      "dstintfrole": { "type": "keyword" },
      "dstip": { "type": "ip" },
      "dstport": { "type": "integer" },
      "duration": { "type": "long" },
      "eventtime": { "type": "date" },
      "level": { "type": "keyword" },
      "logid": { "type": "keyword" },
      "mastersrcmac": { "type": "keyword" },
      "policyid": { "type": "long" },
      "policytype": { "type": "keyword" },
      "poluuid": { "type": "keyword" },
      "proto": { "type": "keyword" },
      "rcvdbyte": { "type": "long" },
      "rcvdpkt": { "type": "long" },
      "sentbyte": { "type": "long" },
      "sentpkt": { "type": "long" },
      "service": { "type": "keyword" },
      "sessionid": { "type": "keyword" },
      "srccountry": { "type": "keyword" },
      "srcintf": { "type": "keyword" },
      "srcintfrole": { "type": "keyword" },
      "srcip": { "type": "ip" },
      "srcmac": { "type": "keyword" },
      "srcport": { "type": "integer" },
      "subtype": { "type": "keyword" },
      "trandisp": { "type": "keyword" },
      "transip": { "type": "ip" },
      "transport": { "type": "integer" },
      "type": { "type": "keyword" },
      "tz": { "type": "keyword" },
      "vd": { "type": "keyword" }
    }
  }
}'
```

2. 새로운 인덱스 생성

```bash
curl -X PUT "http://localhost:9200/fortigate-logs-2025.01.10"
```

3. 기존 데이터 재색인(옵션: 필요 시)

```bash
curl -X POST "http://localhost:9200/_reindex" -H 'Content-Type: application/json' -d '
{
  "source": { "index": "fortigate-logs-old" },
  "dest": { "index": "fortigate-logs-new" }
}'
```

### 최종 확인
1. 템플릿 조회

```bash
curl -X GET "http://localhost:9200/_template/fortigate-logs-template?pretty"
```

2. 인덱스 매핑 확인

```bash
curl -X GET "http://localhost:9200/fortigate-logs-2025.01.10/_mapping?pretty"
```

3. 재색인 상태 확인

```bash
curl -X GET "http://localhost:9200/_tasks?pretty"
```

## 참조
> - [JVM 버전 호환](https://www.elastic.co/kr/support/matrix#matrix_jvm)
> - [logstash 공식문서](https://www.elastic.co/guide/en/beats/filebeat/current/logstash-output.html)
> - [filebeat Processors 공식문서](https://www.elastic.co/guide/en/beats/filebeat/current/filtering-and-enhancing-data.html)
> - [filebeat Logging 공식문서](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-logging.html)

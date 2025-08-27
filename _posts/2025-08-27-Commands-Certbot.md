---
title: Certbot SSL 인증서 발급 방법
author: G.G
date: 2025-08-27 21:10 +0900
categories: [Blog, Command]
tags: [Command, Certbot]
---

## 📘 개요 (Overview)
Certbot은 Let's Encrypt에서 제공하는 무료 SSL/TLS 인증서 발급 자동화 도구입니다. 웹 서버에 설치하여 도메인 소유권을 자동으로 확인하고, SSL 인증서를 발급 및 갱신해주는 핵심적인 역할을 수행합니다. 이를 통해 웹사이트의 HTTPS 통신을 손쉽게 구현하여 데이터 암호화와 보안을 강화할 수 있습니다.

## 📌 목적 (Purpose)
- 🧭 등장 배경: 현대 웹 환경에서 SSL/TLS 인증서는 웹사이트의 보안과 사용자 신뢰를 위해 필수적입니다. 하지만 유료 인증서를 구매하거나 복잡한 수동 설정을 해야 하는 어려움이 있었습니다. 이에 Let's Encrypt는 무료 인증서를 제공하며, Certbot은 이 과정을 자동화하여 누구나 쉽게 HTTPS를 적용할 수 있도록 만들어졌습니다.

- 🎯 기대 효과: Certbot을 사용하면 웹 서버에 무료로 SSL 인증서를 발급하고 적용할 수 있습니다. 또한, Certbot이 인증서 갱신을 자동으로 처리해주기 때문에 인증서 만료로 인한 서비스 중단 걱정을 줄이고, 전반적인 보안을 강화할 수 있습니다.


## 📝 구성 요소 (Components)

| 역할 | 설명 | 특징 |
|---|---|---|
| **Certbot Client** | Certbot의 핵심 CLI(명령줄 인터페이스) 도구입니다. | SSL 인증서 발급, 갱신, 폐기 등의 작업을 수행하는 메인 프로그램입니다. |
| **인증 플러그인 (Authentication Plugin)** | 도메인 소유권을 확인하는 역할을 담당합니다. | **HTTP-01**, **DNS-01** 등 다양한 챌린지 방식을 지원하며, 각 웹 서버나 DNS 서비스에 맞는 플러그인을 사용합니다. |
| **설치 플러그인 (Installer Plugin)** | 발급받은 인증서를 웹 서버에 적용하는 역할을 합니다. | **Apache**, **Nginx** 등 특정 웹 서버의 설정 파일을 자동으로 수정하여 HTTPS를 활성화합니다. |
| **Let's Encrypt CA (Certificate Authority)** | SSL 인증서를 발급해주는 **인증 기관**입니다. | Certbot이 보낸 도메인 소유권 증명을 확인한 후, 실제 인증서를 발급해주는 신뢰 기관입니다. |

## ⚙️ 설지방법

- 패키지 설치

```bash
sudo install -m 600 /dev/null /data/letsencrypt/cloudflare.ini
cat << EOF > /data/letsencrypt/cloudflare.ini
2Hn9DhMCzd8gP-jxXi23IYu0Jx2McXJ18kmkkY3k
EOF
```

- Cloudflare 인증서 발급

```bash
cat << 'EOF' > Certificate.sh
#!/usr/bin/env bash

# Cloudflare API 토큰 파일 경로 (보안을 위해 권한을 600으로 설정하세요)
CF_CREDENTIALS="/data/letsencrypt/cloudflare.ini"

# 사용자로부터 도메인 이름 입력 받기
read -p "도메인 이름을 입력하세요 (예: yourdomain.com): " DOMAIN_NAME
WILDCARD_DOMAIN="*.$DOMAIN_NAME"
EMAIL_ADDRESS="it@$DOMAIN_NAME"

# 사용자에게 선택지를 보여줍니다.
echo "원하는 작업을 선택하세요:"
echo "1) SSL 인증서 발급 (certonly)"
echo "2) 인증서 갱신 (renew)"
echo "3) 인증서 폐기 (revoke)"
read -p "번호를 입력하세요: " choice

case $choice in
    1)
        echo "SSL 인증서를 발급합니다..."
        sudo certbot certonly \
            --dns-cloudflare \
            --dns-cloudflare-credentials $CF_CREDENTIALS \
            --non-interactive \
            --agree-tos \
            --preferred-challenges dns-01 \
            --dns-cloudflare-propagation-seconds 60 \
            -m $EMAIL_ADDRESS \
            -d $DOMAIN_NAME \
            -d $WILDCARD_DOMAIN
        echo "SSL 인증서 발급이 완료되었습니다."
        ;;
    2)
        echo "SSL 인증서를 갱신합니다..."
        sudo certbot renew --dry-run
        # 실제로 갱신할 때는 --dry-run을 제외하고 사용하세요.
        # sudo certbot renew
        echo "SSL 인증서 갱신이 완료되었습니다."
        ;;
    3)
        echo "SSL 인증서를 폐기합니다..."
        sudo certbot revoke --cert-name $DOMAIN_NAME
        echo "SSL 인증서 폐기가 완료되었습니다."
        ;;
    *)
        echo "잘못된 선택입니다. 1, 2, 3 중 하나를 입력하세요."
        exit 1
        ;;
esac
EOF
```

```bash
bash Certificate.sh
```

## 참고 자료
- [Certbot 문서](https://certbot-dns-cloudflare.readthedocs.io/)

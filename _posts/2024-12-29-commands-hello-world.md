---
title: Hello World
description: >-
  안녕하세요, G.G입니다! 이 블로그에서는 제가 배우고 경험한 내용을 공유합니다. 기존 Just-the-docs Jekyll 템플릿에서 작성된 새로운 Chirpy Jekyll 템플릿으로 리뉴얼 합니다.
author: G.G
date: 2024-12-29 22:28:00 +0900
categories: [Blog, Tutorial]
tags: [Form]
pin: true
---

## 📘 소개

> 안녕하세요! G.G의 블로그입니다.
{: .prompt-tip }

## GitHub에서 Chirpy Jekyll 템플릿으로 블로그 시작하기

> Chirpy Jekyll 템플릿 블로그 생성

1. 새로운 저장소 생성
2. 새로운 저장소 등록

![Chirpy](/assets/img/2024-12-29/template.png)
_Chirpy template_
![Create](/assets/img/2024-12-29/create.png)
_Create repository_

## `_config.yml`{: .filepath} 기본 설정

> 언어, 국가별 시간, 블로그 제목, 자신의 닉네임, 아바타를 지정합니다.

```yaml
lang: ko-KR
timezone: Asia/Seoul
title: G.G
name: G.G
avatar: /assets/img/g.g.png
```
{: file='_config.yml'}

| **항목**                 | **설명**                                     | **예시**                 |
|:-------------------------|:---------------------------------------------|--------------------------|
| `lang`                   | (`_data/locales/`{: .filepath}에서 확인 가능) | `ko-KR` (한국어)         |
| `timezone`               | 사이트의 시간대를 설정합니다.                  | `Asia/Seoul` (한국 시간대)|
| `title`                  | 블로그의 제목을 설정합니다.                    | `G.G` (블로그 이름)      |
| social:<br>  `name`      | 작성자의 이름을 설정합니다.                    | `G.G`                   |
| avatar                   | 아바타 사진을 설정합니다.                      | `/assets/img/g.g.png`{: .filepath} |

## `_data/contact.yml`{: .filepath} 기본 설정

> 사용하지 않는 불필요한 SNS 계정정보 제거한다.

```yaml
# - type: twitter
#   icon: "fa-brands fa-x-twitter"

# - type: email
#   icon: "fas fa-envelope"
#   noblank: true # open link in current tab

# - type: rss
#   icon: "fas fa-rss"
#   noblank: true
```
{: file='contact.yml'}

## 참조
- [Jekyll 공식 문서](https://jekyllrb.com/docs/installation/)
- [Chirpy 테마 GitHub 리포지토리](https://github.com/cotes2020/jekyll-theme-chirpy)

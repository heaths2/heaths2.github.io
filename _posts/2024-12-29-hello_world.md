---
title: Hello World
description: >-
  안녕하세요, G.G입니다! 이 블로그에서는 제가 배우고 경험한 내용을 공유합니다.
author: G.G
date: 2024-12-29 22:28:00 +0900
categories: [Blog, Tutorial]
tags: [form]
pin: true
---

## 소개

안녕하세요! G.G의 블로그입니다.

## GitHub에서 Chirpy Jekyll 템플릿으로 블로그 시작하기

![image](https://github.com/user-attachments/assets/7e77c4a8-289f-462d-97fc-71cc92c225cc)
![image](https://github.com/user-attachments/assets/3d26ee42-9fd9-4923-937a-2922b844224a)

## `_config.yml` 기본 설정

```bash
lang: ko-KR
timezone: Asia/Seoul
title: G.G
url: "https://heaths2.github.io"
name: G.G
```

{: .prompt-info }
>
| **항목**    | **설명**                                     | **예시**                 |
|-------------|----------------------------------------------|--------------------------|
| `lang`      | 사이트의 언어를 설정합니다.                  | `ko-KR` (한국어)         |
| `timezone`  | 사이트의 시간대를 설정합니다.                | `Asia/Seoul` (한국 시간대)|
| `title`     | 블로그의 제목을 설정합니다.                  | `G.G` (블로그 이름)      |
| `url`       | GitHub Pages에서 블로그가 배포될 URL을 설정합니다. | `https://heaths2.github.io` |
| `name`      | 작성자의 이름을 설정합니다.                  | `G.G`                   |


## Ruby 및 필수 패키지 설치

```bash
sudo apt-get update
sudo apt-get install ruby-full build-essential zlib1g-dev
```

## Ruby와 Gem의 설치 확인

```bash
ruby --version
gem --version
```

## 환경변수 설정

```bash
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## Jekyll 및 Bundler 설치

```bash
gem install jekyll bundler
```

## Node.js 설치

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 22
```

## Node.js 설치 확인

```bash
node -v
nvm current
npm -v
```

## Jekyll 테마 설치 및 초기화

```bash
git clone https://github.com/cotes2020/jekyll-theme-chirpy.git
cd jekyll-theme-chirpy
bash tools/init.sh
```

## Gemfile에 정의된 모든 종속성을 설치

```bash
bundle install
```

## Jekyll 로컬 서버를 실행
```bash
bundle exec jekyll serve --host 0.0.0.0
```

## 참조
- [Jekyll 공식 문서](https://jekyllrb.com/docs/installation/)
- [Chirpy 테마 GitHub 리포지토리](https://github.com/cotes2020/jekyll-theme-chirpy)

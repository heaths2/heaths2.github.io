---
title: Kubernetes 설치방법
author: G.G
date: 2025-01-01 08:55:00 +0900
categories: [Blog, Bash Shell]
tags: [bash, shell, script]
---

## 개요
XEN VM 이미지를 병렬처리로 복사하는 스크립트 입니다.

```bash
#!/usr/bin/env bash
set -o xtrace
set -e

list=(
control-node01
control-node02
control-node03
worker-node01 
worker-node02
)

#for i in ${list[@]}; do
#  echo $i
#  if [[ -f /img/$i/"$i"_os.raw ]]; then
#      cp -v /img/$i/"$i"_os.raw /img/$i/"$i"_os.img
#  else
#      echo "File /img/$i/${i}_os.raw does not exist, skipping..."
#  fi
#done

cp_image() {
  local node=$1
  echo "Processing $node"
  
  if [[ -f /img/$node/"${node}_os.raw" ]]; then
      cp -v /img/$node/"${node}_os.raw" /img/$node/"${node}_os.img"
  else
      echo "File /img/$node/${node}_os.raw does not exist, skipping..."
  fi
}

export -f cp_image
printf "%s\n" "${list[@]}" | xargs -n 1 -P 2 -I {} bash -c 'cp_image "$@"' _ {}
set +o xtrace
```

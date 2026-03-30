---
title: git 제목 UTF-8 깨짐
date: 2025-12-01
category: gitstudy
tag: [git,error]
---

파일명이 \354\240\225\353\263… 이런 식으로 이상한 바이트열로 깨져 보이는 문제


Git이 파일명을 octal escape로 출력 중이라 발생. 
Git 설정을 바꿔야 한다.

git config --global core.quotepath false

Git은 기본적으로 “이름이 non-ASCII면”
\354\240\225 같은 octal escape 로 출력함

core.quotepath=false 를 설정하면
그대로 한글 UTF-8 텍스트로 보여 줌.

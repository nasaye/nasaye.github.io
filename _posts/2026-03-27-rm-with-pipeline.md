---
title: rm-with-pipeline
date: 2026-03-27 10:47:34 +09:00
categories: [pwshell]
tags: [pwsh]

---

### 요구

 1. */*test*.md 파일들을 한번에 삭제해야 한다.


### 상황

어제 auto-template 파워쉘 자동화 스크립트를 테스트하면서 너무 많은 test 파일들이 생성됐다.
다행히 이름에 전부 test 가 들어갔기 때문에 cmdlet 명령어를 조합해서 지울 수 있을 것 같다!


### 해결

    fd -t f "test" | For-Each-Object { rm $_ }

fd 는 파워쉘 자체 명령어는 아니라서 스크립트에는 쓰지 말라고 하는데,
처음부터 fd로 시작해서 fd가 너무 편하다.

```.ps1
 fd -t f "test"
Error\2026-03-26-test.md
daily_study\mermaid_test.md
python\2026-03-26-test.md
side_project\auto_template\templates\test.md
test\2026-03-25-test2.md
test\2026-03-25-test3.md
test\2026-03-25-test4.md
test\2026-03-DD-do-test.md
thoughts\2026-03-26-test.md
```
```
 fd -t f "test" | ForEach-Object { rm $_ }
 fd -t f "test"

```
깔끔하게 지워졌다.
스크립트를 쓰기 시작한지 3일 정도 된 것 같은데, 생각보다 파일 관리가 편해진다.

하지만 아직 명령어를 입력했을 때 어떤 결과값이 파이프로 전달되는지는 정확히 모른다.
공부해서 알아봐야겠다.


---

















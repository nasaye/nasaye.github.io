---
title: 메모 템플릿 자동화-3
date: 2026-03-29 21:00:47 +09:00
categories: [automation]
tags: [git,pwsh]
---

### git-page 에 포스트하는 pwsh 함수 만들기

목표는 명료하다. 자동 포스팅.
자동 템플릿 만들기로 포스트를 위한 파일양식을 준비했으니, 
이제 자동으로 업로드하는 일만 남았다.

그런데 생각보다 쉬울 것 같다.

예상 구조:

[카테고리 디렉토리]
- 카테고리 주제별로 메모 작성해서 보관
- git 저장소로 별도로 관리
- 메모는 템플릿 자동 생성

[블로그 레포지토리]
- 카테고리에서 원하는 포스트를 copy 명령어로 가져온다.
- '_post' 디렉터리에 넣는다.
- git commit 으로 커밋한다.
- 나머지는 git action 에서 처리한다.


카피, 커밋 과정만 구현해 주면 쉽게 작동할 것으로 예상됨.

---

### 에러발생 1. 권한 설정 오류

```
 comp
[main e8d47e7] 2026-03-29 21:23:41 blog posting
 1 file changed, 400 insertions(+)
 create mode 100644 _posts/2026-03-28-auto-template-1.md
fatal: 'origin/main' does not appear to be a git repository
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
 git push
remote: Permission to nasaye/nasaye.github.io.git denied to R4dex4.
fatal: unable to access 'https://github.com/nasaye/nasaye.github.io.git/': The requested URL returned error: 403

```

global user 로 등록한 git 아이디랑, git blog 레포지토리를 관리하는데 사용하는 git아이디가 다른 걸 간과했다.
그 결과 push 요청이 denied 되었고, 이럴 땐 권한 부족 접근거부 403 에러가 뜨네.
그럼 하나의 OS 사용자 계정에서 여러개의 git 계정을 관리하는 방법을 익혀 놓아야겠다.

* 학습 완료!



---

### 구현

```
# 딸깍 포스팅 함수

function posting {

	#매개변수로 포스팅할 파일 주소 불러오기
	param(
		[string]$content
	)
	#블로그 저장소 경로와 날짜 변수 설정
	$path = "D:\blog"
	$date = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')"

	# 존재하지 않는 글을 포스팅하려고 할 경우 종료
	if (-not (Test-Path $content)) {
		Write-Host "포스팅하려는 파일이 존재하지 않습니다."
		return
	}
	
	# 포스팅할 글을 저장소의 _posts 로 복사
	Copy-Item $content $path\_posts

	# git 으로 remote 에 push
	git -C $path add .
	git -C $path commit -m "$date blog posting"
	git -C $path push
}
```

구현 완료. 복잡한 내용도 없었다.


### 보완점

1. fzf를 사용해서 파일 클릭으로 바로 포스팅하기

2. 이미 '_posts' 에 파일이 있을 경우의 처리

3. md 파일 확장자 체크




















---

















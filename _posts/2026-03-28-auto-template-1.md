---
title: 메모 템플릿 자동화-1
date: 2026-03-25 10:00:00 +0900
category: [automation]
tag: [automation, pwsh]

---

### 목표 (요구사항)

nvim 툴을 메모장이나 py 코드 잠깐 돌리는데만 사용하고 있는데, 
깃 페이지에 메모했던 내용들을 올리려고 메타데이터를 입력하다 보니
형식을 반복해서 기록해야 하는 작업이 매우 불편했다.

오늘 오전에 1시간정도 시간이 남아서, 자동화에 도전해 보려고 한다.

요구사항:
>    1. 터미널에 note 키워드를 입력해 nvim 을 오픈
>    2. note -[temp] 예약어로 [temp] 템플릿을 가져오기
>    3. 파일의 제목은 gitpage 양식에 맞춰 자동작성
>    4. 프론트매터 항목 자동작성
>    5. note -[temp] -[cat] 카테고리 항목 채우고 기준폴더의 해당 카테고리 폴더에 저장


---
### 1. 터미널에 키워드 입력 구현

우선 pwsh 프로필에 새 함수를 추가해 보자.
alias 별칭 생성으로 note 입력시 nvim 을 띄울 수 있겟지만 다음 요구사항을 설계하기 어려울 것 같다.
함수를 만들어서 사용해 보자.

```profile.ps1
#zoxide (z lib) ... 
# oh my posh ...
# PSFzf 키바인딩 ...
function cdd { 
  $f = fd -H -t f . $env:USERPROFILE | fzf
  if ($f) { cd (Split-Path $f) }
}

```
프로필에 선객들이 좀 있는데, 맨 밑에 함수를 선언해 보자.

 5. note -[temp] -[cat] 카테고리 항목 채우고 기준폴더의 해당 카테고리 폴더에 저장
```profile.ps1
function note {
    nvim
}

```
심플한 구조이지만 잘 작동한다.

- function : 함수 정의 키워드
- note : 함수 이름이 바로 명령어로 사용됨
- { ... } : 스크립트 블록. 함수가 수행할 코드.
    - 스크립트 블록에 입력된 명령들은 호출시 파워쉘 세션 안에서 순서대로 실행됨!
    - 즉 터미널에 쓸 지루하고 현학적인 내용들을 여기에 넣으면 된다.

---
### 2. 템플릿 가져오기

우선 템플릿을 가져오려면 템플릿.md 파일을 만들어야겠지?

gitpage 의 양식을 확인해보자.

1. "_post 의 파일 제목 양식

> 2026-03-06-router-IGP-algorythm.md

[YYYY-MM-DD]-[영어제목].md 형식이다.

2. YAML 프론트매터

```a.md
---
title: "내 글 제목"
date: YYYY-MM-DD
categories: [,]
tags: [,]
---
```

YAML 야멜은 JSON 같은 데이터포맷이고, 
포스트 위에 붙여야해서 귀찮은 메타데이터 블록의 이름은 프론트매터.

임시로 가볍게 first_t.md 를 만들어 보자.

```first_t.md
---
title: "내 글 제목"
date: YYYY-MM-DD
categories: [,]
tags: [,]
---
### 개요
### 요약
---
### 느낀점
```

이제 이 템플릿을 보관할 dir을 만들어야 하는데, 현재 위치에 템플릿 dir을 만들고
그 안에 first_t.md 를 저장하자.

다 완료했다면, 간단한 방법으로 템플릿을 호출할 수 있다.

```PROFILE.ps1
function note {
	nvim D:\...\auto_template\templates\first_t.md
}

```

하지만! 이렇게 호출하면 그냥 지정위치에 있는 md파일을 여는 것에 지나지 않는다.
first_t.md 파일의 내용물만 빼서 우리가 원하는 위치에 새 md 파일을 만들어야 한다.

기존의 nvim -파일위치 는 여는 기능일 뿐이기 때문에,
note 에 대한 새로운 매개변수 형식을 만들어야 한다!

```PROFILE.ps1

function note {
	param(
		[string]$template
	)
	Write-Output "$template"
#   nvim D:\...\auto_template\templates\first_t.md}

```

[MS LEARN PARAMETERS](https://learn.microsoft.com/ko-kr/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters?view=powershell-7.5)

param 은 함수 외부에서 매개변수를 받을 수 있게 해 준다.
Write-Output 으로 string타입 $template 변수로 받은 값을 출력해서 테스트.

- "$" 는, 변수를 나타내는 기호다. 
- Write-Output = $a 처럼 변수의 값을 꺼낼 때도 사용한다.

$template param을 통해 주소값 문자열을 입력받는다면, 
nvim 과 마찬가지로 해당 파일을 여는 것에 지나지 않는다.

그렇다면 템플릿의 내용을 읽어서 문서에 그대로 넣는 방법이 있지 않을까.

```a.ps1
Get-Content

```

위 명령어로 컨텐츠를 읽을 수 있다고 한다. 
그렇다면 nvim 으로 파일을 만들고 Get-Content 와 > 리다이렉션 연산자로 내용을 넣어 보자.

```$PROFILE.ps1

function note {
	param(
	#	[string]$template
		[string]$title
	)
	
	nvim $title.md
	Get-Content D:...\auto_template\templates\first_t.md > $title.md

	}

```

실행되지 않는다. 정확히는 이름이 비어있는 nvim이 열린다.
아마도, nvim 에서 편집 후 w 로 저장하기 전까지는 파일이 생성되지 않았다 보니 Get-Content 가 작동하지 않는 것 아닐까?

그리고 pwsh 명령어를 사용중인데 외부 프로그램이나 다름 없는 nvim 을 쓰는 것도 변수가  될 것 같다. 파워쉘의 파일 생성 명령어를 찾아서 써 보자.

```$PROFILE.ps1

function note {
	param(
	#	[string]$template
		[string]$title
	)
	
    New-Item . $title.md Get-Content D:...\side_project\auto_template\templates\first_t.md

	}

```
어디 내놓기 부끄러운 형편없는 코드가 나왔다..

```error.ps1
New-Item: C:\Users\cynal\Documents\PowerShell\Microsoft.PowerShell_profile.ps1:23
Line |
  23 |      New-Item . $title.md Get-Content D:\velog\velog\side_project\auto …
     |      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     | A positional parameter cannot be found that accepts argument '$null'.

```
New-Item 명령어에 대한 매개변수 형식을 전혀 모르고 있었다.

```.ps1
New-Item -Path . -Name "sample.txt" -ItemType File -Value '내용'

```
이런 매개변수 입력 형식을 가지고 있다.

---
### 2,3,5 번 요구사항을 한번에!

생각해 보니, New-Item 에는 1. 이름 설정, 2. 저장 위치 를 동시에 설정할 수 있다.
그렇다면  요구사항 5번,

> 5. note -[temp] -[cat] 카테고리 항목 채우고 기준폴더의 해당 카테고리 폴더에 저장

의 카테고리 항목을 템플릿과 접합시키는 것도 나쁘지 않겠다!

어차피 나는 메모들을 디렉터리 시스템 하에서 카테고리에 따라 분류할 생각이었다.

그렇다면 템플릿 명을 각 카테고리 명으로 만들고, 
매개변수 란에 $category 변수 값을 입력받은 후,
저장위치는 $path = $base\$category 로,
템플릿은 Get-Content $base\$tempbase\$category.md > $path 로 
설정해주면 되겠다!

이렇게 된다면, 
    2. note -[temp] 예약어로 [temp] 템플릿을 가져오기
    5. note -[temp] -[cat] 카테고리 항목 채우고 기준폴더의 해당 카테고리 폴더에 저장
이 두 항목을 하나로 합칠 수 있겠다!
하는 김에, Get-Date -Format 'yyyy-MM-dd' 를 사용해서 날짜도 제목에 붙여보자!

```
function note {
	param(
		[string]$title,
		[string]$category
	)
	$base = "D:...\"
	
	$path = "$base\$category\$(Get-Date -Format 'yyyy-MM-dd')-$title.md"

	New-Item -ItemType File -Path $path
	Get-Content $base\side_project\auto_template\templates\$category.md > $path

	nvim $path
	}
```
결과는? 성공!
마지막 nvim $path 는 어차피 nvim으로 열어서 작성할 거니까~

---

###  4. 프론트매터 항목 자동작성

이제 나를 항상 귀찮게 했던 원흉! 메인 디쉬 차례다.
기본적인 개념은, 생성한 md 파일의 프론트매터에
플레이스홀더(placeholder)를 놓고,
function note{} 에 Get-Date 등을 이용해서 값을 채워넣는 것이다.만,

할 줄 모른다!
모르는 걸 해낼 순 없으니까 찾아서 해보자.

#### 플레이스홀더란?

파워쉘과 하는 바꿔치기 약속!
우리가 기존에 지정해놓은 특정 문자열을 찾아서 약속된 내용으로 바꾸는 것!

이렇게 생각하니까 간단하다.
변수 a에 Get-Content $path cmdlet 을 사용해 데이터값을 넣고, 
a.replace("<치환하기로 한 키워드>", 치환값)
을 해 주면 값이 치환되는 거다.
그 후 Set-Content 로 원래 있던 자리로 돌려보내 주면 되겠다.

---

자, 해보자.
우선 템플릿 md파일에 플레이스홀더를 넣어주자.


```test.md
---
title: {{title}}
date: {{date}}
categories: [{{category}}]
tags: [,]
---
### 개요
### 요약
---
### 느낀점
```
그럼 이제 파워쉘 프로필에서 함수를 수정해 보자.

```PROFILE.ps1

function note {
	param(
		[string]$category,
		[string]$title
	)
	$base = "D:...\"
	$path = "$base\$category\$(Get-Date -Format 'yyyy-MM-dd')-$title.md"
	$date = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') $(Get-Date -Format 'zzz')"

	New-Item -ItemType File -Path $path
	$content = Get-Content $base\side_project\auto_template\templates\$category.md

	$content = $content.Replace("{{title}}", $title)
	$content = $content.Replace("{{date}}", $date)
	$content = $content.Replace("{{category}}", $category)
	
	Set-Content $path $content

	nvim $path
	
	}

```

결과는, ~ 성공!

```2026-03-25-test3.md
---
title: test3
date: 2026-03-25 23:30:17 +09:00
categories: [test]
tags: [,]
---
### 개요
### 요약
---
### 느낀점
```


### 피드백

그런데, Get-Content 의 내부 동작 원리는 
파일 열기 -> 한 줄씩 읽기 -> 배열로 반환
이라고 한다. 그래서 플레이스홀더를 대체할 때, 순수하게 문자열로 읽는 -Raw 옵션을 켜 주는게 좋대.

추가로, 어디에 있든지 note 함수를 통해 메모를 작성하고 나면 편집을 종료했을 때
해당 카테고리에 가 있으면 좋겠네.
함수 내에서는 cd cmdlet 을 사용해서 세션 이동이 가능해.

```최종1.ps1
function note {
	param(
		[string]$category,
		[string]$title
	)
	$base = "D:\velog\velog"
	$path = "$base\$category\$(Get-Date -Format 'yyyy-MM-dd')-$title.md"
	$date = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') $(Get-Date -Format 'zzz')"

	New-Item -ItemType File -Path $path
	$content = Get-Content -Raw $base\side_project\auto_template\templates\$category.md

	$content = $content.Replace("{{title}}", $title)
	$content = $content.Replace("{{date}}", $date)
	$content = $content.Replace("{{category}}", $category)
	
	Set-Content $path $content
	
	cd "$base\$category"
	nvim $path
	}

```

오늘은 여기까지!


### 다음 목표 (안정화)



1. New-Item 이미 있을 경우에 대한 처리 프로세스

```
Line |
  26 |      New-Item -ItemType File -Path $path
     |      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     | The file 'D:\velog\velog\test\2026-03-25-firsten.md' already exists.

```
 질문을 띄우고, 이름을 바꿀 건지 덮어쓸 건지 선택하게 하는 게 좋겠어.


2. 카테고리 입력에 잘못된 형식이 들어올 경우의 처리

카테고리 리스트에 없는 항목을 무작정 입력할 경우 지금은 그냥 nvim이 초기화 상태으로 열린다. 
항목에 없는 입력은 차단하도록 만들어야겠지?


3. 카테고리에 따라 템플릿 리스트 동기화, 자동 생성

필요하다고 봐.

4. title 에 공백이 있을 경우의 처리.

공백이 있을 경우엔 git page 오류가 날 수 있음. 





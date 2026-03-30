---
title: 메모 템플릿 자동화-2
date: 2026-03-26 19:08:11 +09:00
categories: [automation]
tags: [automation,pwsh]
---

메모 템플릿 자동화-1 의 코드를 개선해보자.


### 요구사항

1. 디렉터리의 상태에 맞춰 카테고리 리스트 관리하기
2. 카테고리 명에 맞는 템플릿이 없을 경우 템플릿 생성
3. 파일 생성시 이름이 중복되는 상황에서 생성 중지
4. 카테고리 명이 잘못되었을 경우 재입력 또는 중지

### 1. 카테고리 리스트 관리

일단, 디렉터리 리스트를 긁어오자.
리다이렉션 cmdlet 를 사용하면 할 수 있을 것 같다.

    fd -t directory > dirlist.md

```dirlist.md
Error\
IS\
IT\
book_review\
daily_study\
diary\
gitstudy\
memo\
python\
security\
side_project\
side_project\auto_template\
side_project\auto_template\templates\
term\
test\
thoughts\
```
긁어오기 성공!

이제 디렉터리(카테고리)에 붙어있는 역슬래시를 파싱해 보자.
우선 긁어온 데이터를 변수에 넣어 보자. 

    $cat_list = fd -s -t directory . $base
    $cat_list.GetType()

일단 변수에 넣는데 까진 성공했다.
그런데 문제가 조금 있다.

```.ps1
IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array

```
일단, 저장한 내용이 Array 타입이다.
fd 의 결과값이 array 타입인듯. -raw 키워드를 찾아봐야겠다.

```.ps1
fd -0 -s -t directory . 'D:~~'

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     String                                   System.Object

```
-0 라는 플래그를 사용했더니 String 이 되긴 했다.

그 다음으론 루트주소를 지우고 파일명만 남기는 것과,
fd 가 디렉터리 내부구조까지 참조하지 않는 것을 구현해 보자.

    fd -0 -s -t directory --max-depth = 1 . 'D:\velog\velog'

max-depth 옵션으로 디렉터리 내부 표시는 감췄다.
이제 주소 지우기를 해 보자.


...

그냥 배열로 처리하는게 더 편하겠다!
파워쉘에서는 각 타입에 대한 처리를 원활하게 할 수 있으니까.

    Split-path 

라는 cmdlet 가 있다.
경로의 부모와 자식 중 일부분만 지정해서 문자열로 반환한다.
-leaf 플래그를 붙이면, 자식 부분만 분해하는 것도 가능하다.



[Split-path](https://learn.microsoft.com/ko-kr/powershell/module/microsoft.powershell.management/split-path?view=powershell-7.5)

`ForEach-Object` 는 배열 객체를 하나씩 순회하면서 지정된 기능을 적용해 준다.
`$_` 는 배열 객체의 각각의 요소를 나타내는 임시 변수라고 보면 된다.

`$category_list = fd -s -t directory . $base | ForEach-Object { Split-Path $_ -Leaf }`


결과 
```
Error
IS
IT
book_review
daily_study
diary
gitstudy
memo
python
security
side_project
auto_template
templates
term
test
thoughts

```

성공. 이제 디렉터리에 어떤 카테고리가 있는지 확인할 수 있게 되었다.

### 2. 카테고리 명에 맞는 템플릿이 없을 경우 템플릿 생성

템플릿이 없을 경우 아래의 에러가 발생한다.

```
Get-Content: C:\Users\cynal\Documents\PowerShell\Microsoft.PowerShell_profile.ps1:47
Line |
  47 |  …  $content = Get-Content -Raw $base\side_project\auto_template\templat …
     |                ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     | Cannot find path 'D:\...\templates\python.md'
     | because it does not exist.
InvalidOperation: C:\...\Microsoft.PowerShell_profile.ps1
```
입력받은 카테고리명을 템플릿 저장 주소 뒤에 붙여서, Test-Path cmdlet로 템플릿이 있는지 확인한다.
없을 경우, 기본 템플릿을 복사, 이름을 카테고리명으로 변경해서 템플릿을 생성한다.

	# 해당 템플릿이 없을 경우 기본 템플릿 생성
	if (-not (Test-Path $template_path\$category.md)) {
		Write-Host "$category 에 대한 기본 템플릿을 생성합니다."
		Copy-Item $template_path\base_template.md $template_path\$category.md
		Write-Host "생성 완료."
		}

여기에다가 `Start-Sleep` cmdlet 를 사용해 볼까 고민했는데,
굳이 시간을 지연시킬 필요까진 없어 보여 넣지 않았다.


### 3. 파일 생성시 이름이 중복되는 상황에서 생성 중지

이건 크게 당하고 나서야 넣게 된 기능이다...
중복 파일명으로 Set-Content 를 하니 기존 파일이 사라졌다.

	if (Test-Path $path) {
		Write-Host "$title 파일이 이미 존재합니다.`n 생성을 중지합니다."
		cd "$base\$category"
		nvim $path
		return
	}

설명한 필요 없는 간단한 코드다.
그냥 닫는 대신 어떤 파일이 있었는지 열어보게 해 놨다.


### 4. 카테고리 명이 잘못되었을 경우 재입력 또는 중지

없는 카테고리를 입력했을 때 문제가 발생하기 때문에 재입력을 받기로 했다.
처음에는 그냥 종료하려고 했는데, while 문을 써서 재입력을 받는 게 내 입장에서 더 편했다.
거기다가 한 번 틀렸을 때 카테고리 리스트를 보여주면 그걸 보고 칠 수 있도록 짜 봤다.

```
	while (-not($category_list -contains $category)) {
		if ($category -eq "exit") {
			return
		}
		Write-Host "카테고리가 없습니다.`n 카테고리 리스트:"
		$category_list
		$category = Read-Host "(카테고리를 다시 입력해 주세요. (종료:exit 입력)  "
	}

```
`A -contains B` 는 A 객체의 요소 안에 B가 들어있으면 True 를 반환한다.
`-eq` 는 c언어의 == 비교연산자처럼 사용하면 된다. 완전일치여부를 판단한다.
터미널에서 다시 입력을 받고 싶다면, `Read-Host` cmdlet 를 사용하자.


...

위의 while 문에는 큰 이상이 없어 보였는데, 또 자꾸만 에러가 났다.

```
    Set-Content: C:\Users\cynal\Documents\PowerShell\Microsoft.PowerShell_profile.ps1:54
Line |
  54 |      Set-Content $path $content
     |      ~~~~~~~~~~~~~~~~~~~~~~~~~~
     | Could not find a part of the path 'D:\velog\velog\dive\2026-03-26-das.md'.


```

문제의 원인은 처음 선언한 `$path` 변수였다.
함수의 최상단에서 

	$path = "$base\$category\$(Get-Date -Format 'yyyy-MM-dd')-$title.md"

이렇게 선언되고 있는데,
그 이후에 while문에서 $category 를 재입력 받았으니...
while 문을 통과해도 $path 는 갱신되지 않아 사용할 수 없었다.
단순한 문제였지만 찾는 데 꽤 시간이 걸렸다. 앞으로 변수 값에 변수가 포함되는 경우 이 경험을 토대로 실수하지 말아야겠다.

```
	while (-not($category_list -contains $category)) {
		if ($category -eq "exit") {
			return
		}
		Write-Host "카테고리가 없습니다.`n 카테고리 리스트:"
		$category_list
		$category = Read-Host "(카테고리를 다시 입력해 주세요. (종료:exit 입력)  "
	# 카테고리 변수 값이 변경되었을 경우, 그 값을 사용하는 $path 변수도 갱신되어야 한다.
		$path = "$base\$category\$(Get-Date -Format 'yyyy-MM-dd')-$title.md"
	}

```
`$category = Read-Host` 때마다 
`$path = "$base\$category\$(Get-Date -Format 'yyyy-MM-dd')-$title.md"` 를 다시 해줌으로써 오류를 바로잡았다.


### 완성본

```
# 메모 템플릿 자동화 함수
function note {
	
	# 터미널 입력 매개변수 설정
	param(
		[string]$category,
		[string]$title
	)

	# 각종 주소 설정
	$base = "D:\@@"
	$path = "$base\$category\$(Get-Date -Format 'yyyy-MM-dd')-$title.md"
	$template_path = "$base\@@\templates"

	# Get-Date 로 날짜 객체 받아오기
	$date = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') $(Get-Date -Format 'zzz')"
	
	# $base dir 의 폴더명 파싱해서 저장하기
	$category_list = fd -s -t directory . $base |
    	ForEach-Object { Split-Path $_ -Leaf }

	# 입력 받은 카테고리가 존재하지 않을 경우 재입력 유도
	while (-not($category_list -contains $category)) {
		if ($category -eq "exit") {
			return
		}
		Write-Host "카테고리가 없습니다.`n 카테고리 리스트:"
		$category_list
		$category = Read-Host "(카테고리를 다시 입력해 주세요. (종료:exit 입력)  "
	# 카테고리 변수 값이 변경되었을 경우, 그 값을 사용하는 $path 변수도 갱신되어야 한다.
		$path = "$base\$category\$(Get-Date -Format 'yyyy-MM-dd')-$title.md"
	}

	# 해당 템플릿이 없을 경우 기본 템플릿 생성
	if (-not (Test-Path $template_path\$category.md)) {
		Write-Host "$category 에 대한 기본 템플릿을 생성합니다."
		Copy-Item $template_path\base_template.md $template_path\$category.md
		Write-Host "생성 완료."
		}

	# 같은 파일명을 가진 파일이 이미 존재할 경우 생성을 중지한다.
	if (Test-Path $path) {
		Write-Host "$title 파일이 이미 존재합니다.`n 생성을 중지합니다."
		cd "$base\$category"
		nvim $path
		return
	}

	# 템플릿의 플레이스홀더에 .Replace 를 사용해 입력정보를 자동화한다.
	$content = Get-Content -Raw $template_path\$category.md
	$content = $content.Replace("{{title}}", $title)
	$content = $content.Replace("{{date}}", $date)
	$content = $content.Replace("{{category}}", $category)
	
	# 
	Set-Content $path $content
	
	cd "$base\$category"
	nvim $path
}
```

기능적으로 불편하지는 않은 함수가 완성됐다.
나름 자동화라고 할 수 있을까? 포스팅을 올리는 속도가 빨라졌다.
다만 아직 개선해야할 점이 많으니, 시간이 날 때마다 하나씩 건드려 보자.


### 개선점

1. 파일명 예외 처리는 다음에 하자.
2. nvim에 대한 결합도가 너무 높으니, nvim을 변수에 할당하고 맨 위에서 사용 프로그램을 바꿀 수 있도록 해야겠다. 










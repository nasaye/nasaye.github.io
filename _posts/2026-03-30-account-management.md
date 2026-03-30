---
title: git 다중 계정 관리법
date: 2026-03-30 10:58:43 +09:00
categories: [gitstudy]
tags: [git]

---

### 개요 : Permision denied

난 깃허브 계정을 2 개 가지고 있고, 한 OS 사용자 환경에서 동시에 운용하려고 한다.
각 계정이 서로 다른 리포지토리를 관리하고 있다면, 원격 push할 때 충돌이 발생할 가능성이 있다.

```
 git push
remote: Permission to ~~~.io.git denied to A.
fatal: unable to access 'https://~~~.io.git/': The requested URL returned error: 403

```
이 문제를 해결하기 위해선 어떻게 계정을 관리해야 할까?

### 1. 현 git 설정 상황 확인하기

    git config --list --show-origin

위 명령어를 통해 지금 내 git의 설정 상황을 파악하자.

```
...
user.name
user.email
http.schannelcheckrevoke
http.sslverify

```
명령을 통해 해당 설정의 설정값들을 확인할 수 있다.
이 항목들의 시작 주소가 C: 부터 나오면 전역설정이라고 보면 된다.
내 계정의 경우 전역설정에 user.name, user.email이 설정되어 있다.
그렇기 때문에 전역 계정 A가 권한이 없는 B의 레파지토리에서 Push하는 문제가 발생했다. 


### 2. Local Repository 레벨에서의 관리: local 설정

global 계정의 문제점은 여러 계정이 한 OS 사용자 안에 존재할 때, 
누가 어떤 활동을 했는지 확인할 수 없다는 것이다. 
또, 방금처럼 다른 local 레퍼지토리에 global로 접근해버리는 문제도 있다.
그렇기 때문에 Local 수준에서는 지역 계정 설정을 사용하는 것이 표준적이다.

일단 전역 설정을 해제해 보자.

    git config --global --unset user.name
    git config --global --unset user.email

위와 같이 했을 때, 
git config 에서 이름과 이메일 항목이 사라진 걸 확인할 수 있다.

이제 각 저장소별로 로컬 계정을 설정해 보자.

    git config --local user.name ""
    git config --local user.email ""
    git config --list

나열된 설정에 user.name, email 항목이 잘 들어가 있다면 성공이다.

#### 2.5 git clone Repository 

난 B의 저장소를 클론해 왔다면 B로만 관리하고 싶은데,
실수로 A의 계정으로 B의 저장소를 클론해 왔단 사실을 알게 됐다.
그럴 땐 local repo를 과감하게 지우고,

    git clone "https://[원하는 계정]@주소" "원하는 디렉터리명"

위의 코드를 입력해주면 된다. 
HTTPS 를 사용하는 방법 말고, SSH 를 사용하는 방법도 있는데,
바로 아래에서 SSH 설정하는 법을 알아볼 예정이라 넘어가겠다.

### 3. ssh 설정

SSH란, 네트워크상 다른 컴퓨터에 안전하게 접속하기 위한 통신 프로토콜이다.
전송되는 모든 데이터를 암호화하고, 해킹, 스니핑 등을 방지한다.
SSH를 통해 암호화된 터널이 생성되고, 그 터널을 통해 데이터가 안전하게 전달된다.

git으로 원격 서버와 데이터를 교환할 때에도 이 SSH를 사용하는 옵션이 있다.
직접 해본 결과, 구조는 이런 식이다.

1. 사용자는 이메일 주소를 기준으로 클라이언트에 SSH 인증키를 발급받는다.

    ssh-keygen -t ed25519 -C "cynalline@gmail.com" -f .ssh/id_rsa_R4dex

ssh-keygen 명령어를 사용하면 -t 로 타입을 설정할 수 있다.
-f 로는 파일 저장위치와 파일 명을 설정할 수 있다.
ed25519는 키를 생성할 때 사용하는 암호화 알고리즘.

2. 키를 발급받으면 사용자가 보관해야 할 비밀키와, 연결할 서버에 제공할 공개키(pub)가 생성된다.

3. 사용자는 서버에(github) 공개키를 제공한다.

4. 디지털 서명을 통한 인증 과정이 실행된다.
    - 사용자가 접속 시도시 서버가 난수를 보냄
    - 사용자는 비밀키로 난수에 서명
    - 서버가 사용자의 공개키로 서명 확인.


아래의 내용은 깃허브에 내 계정의 SSH의 공개키를 등록하지 않아서 발생한 에러다.

```
 ssh -T git@github.com-[계정명]
git@github.com: Permission denied (publickey).

```

SSH 공개키를 등록하려면 github에 로그인하고, 프로필-설정-SSH and GPG keys 에서
New SSH key 로 인증 키를 등록하면 된다.

#### 3.5. ssh config 설정으로 여러개의 인증키 관리

ssh 키가 저장된 .ssh 디렉터리에, config 파일을 생성한다.
확장자가 없는 순수한 config 다.

config 파일에 인증키의 이름을 등록해 놓으면 git 주소 호출시 이름을 같이 입력하는 것으로 원하는 인증키를 골라서 전송할 수 있다.

    git clone git@github.com-[추가로 인증키 이름 입력]:~/~.github.io.git

```
# 1번째 인증키 A
Host github.com-A
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_A

# 2번째 인증키 B
Host github.com-B
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_B

```

위의 config 설정이 동작하는 원리는 이렇다.
SSH 프로토콜은 git 이 원격 서버(Remote)와 통신할 때 주로 사용된다.

1. Git이 git@github.com 으로 시작하는 주소를 보면, SSH 연결임을 파악하고 SSH를 호출한다.

 여기서 git@github.com 은 git 에 접속하기 위한 Common User 아이디다.
 SSH로 git에 접속할 땐 무조건 git이라는 이름으로 접속한다.
 서버가 수천만 명의 사용자에게 각각 리눅스 시스템 계정을 만들기 어렵기 때문에,
 git@ 을 대표계정으로 설정해 두고 들어오는 사람의 SSH 키만 식별하는 방식이다.

2. SSH는 주소를 보고 설정파일 .ssh/config를 읽는다.
 
 기존 주소가 git@github.com:주소 였다면, 
 원하는 인증키를 선택할땐 git@github.com-인증키 이름:주소 경로를 사용한다.
 SSH는config에서 'git@github.com-인증키이름' 이 Host에 존재하는지 보고,
 존재한다면 그 Host에 있는 내용들을 적용시킨다.

 주소 : github.com
 사용자명 : git
 키 : ~/.ssh/id_rsa_A

3. SSH 가 서버인증 및 통로 개설은 완료한다.

4. Git 이 데이터를 전송한다.


`git config --list`

명령어를 이용해 내 원격 Git 주소 명칭이 어떻게 설정되어 있는지 확인한다.

`remote.origin.url=git@github.com-A:A/A.github.io.git`

이렇게 com-A: 로 설정되어 있으면 이 저장소에서 발생하는 커밋, 푸쉬는 기본적으로 A 인증키를 사용해서 실행된다.

이걸 B로 바꾸고 싶다면,

`git remote set-url origin git@github.com-B:A/A.github.io.git`

이렇게 -인증키 이름 만 바꿔주면 된다.

### 결론

한 사용자 환경에서 여러개의 Git 계정을, local 설정과 SSH를 사용해서 구분하는 법을 익혀 보았다.

그런데 보통은 사용자를 다르게 한단다. 그게 확실히 편해 보이기도 한다. 바꿀까...

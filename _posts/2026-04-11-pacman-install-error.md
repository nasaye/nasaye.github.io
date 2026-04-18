---
title: pacman-install-error
date: 2026-04-11 23:50:11 +09:00
categories: [error]
tags: [linux]

---
### 발생

pacman 사용 중 설치 또는 업데이트 과정에서 강제 종료를 하면
패키지 데이터베이스가 잠긴 상태로 남을 수 있다.

이 경우 다음과 같은 에러가 발생한다:

    error: failed to init transaction (unable to lock database)
    error: could not lock database: File exists

/var/lib/pacman/db.lck 파일이 남아 있어, 
pacman이 다른 프로세스가 실행 중이라고 판단하기 때문이다.

### 해결

현재 pacman 프로세스가 실행 중이지 않은 것을 확인한 후,
잠금 파일을 수동으로 삭제한다.

    rm /var/lib/pacman/db.lck

다시 pacman 명령어를 실행하면 정상적으로 동작한다.

### 주의

pacman이 실행 중인 상태에서 db.lck를 삭제하면 데이터베이스 손상 위험이 있다.
반드시 ps aux | grep pacman 등으로 실행 여부를 확인 후 삭제할 것



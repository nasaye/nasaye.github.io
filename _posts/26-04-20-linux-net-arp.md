---
title : ARP NUD 의 상태 관리 매커니즘에 관하여 
date : 2026-04-20 16:15:00 +0900
category : [study,network]
tag : network, study, linux, nud, error
---


### 1. wireshark 기본 형식 패턴 파악

NO.       로그의 일련번호

Time      로그 시작으로부터 초 단위 기록

Source    송신 IP or MAC

Dest      타겟 IP or MAC

Protocol  프로토콜 종류

Length    프레임 길이(byte)  <--- L2 레이어 패킷 감시이므로 프레임 확인

Info      프레임 정보


### 2. ARP 2번 반복 현상

상황 : 

수업 때 배운 것과 다른 형태의 ARP 교환 발견


![ARP로그](https://github.com/user-attachments/assets/3799435c-b03b-4d63-b018-9dc5fa287466)


4.4.4.2 를 a, 4.4.4.3 을 b로 지칭한다.

ARP는 송신자 a가 수신자 b 의 ip주소만 알고, MAC addr 을 모를 때 자동으로 실행.

그런데 a의 ARP request 를 b가 수신하고, b가 ARP reply 를 보낸 후에 다시 b 가 a에게 ARP request 를 보내는 상황이 반복.

a의 ARP req 에는 a의 MAC addr 이 존재하므로 굳이 b->a ARP request 할 이유가 없다고 생각함.

조사 :

단서 1. 90번 ARP Request 로그의 Dest 가 broadcast 인 반면, 99번 역 ARP Request 는 이미 VM_d6:e8:43으로 MAC 주소를 알고 있음.

단서 2. 91번 ARP reply 의 로그 발생 후 약 5.5 초가 지나고서야 b의 ARP Request 가 발생.


즉 b->a ARP Repuest 는 b가 a의 MAC 을 이미 알고 있는 상태에서 진행.


### 3. NUD (Neighbor Unreachability Detection)

문제의 핵심은 NUD 에 있었다.

NUD 란 이웃(IP) 의 MAC 가 실제 도달 가능한지, 유효한지를 검증하는 시스템.

ARP 테이블은 시간 경과에 따라 상태가 변함 (장비가 꺼지거나 네트워크 이동 등)

그래서 테이블 데이터의 유효성을 지속적으로 검사.

  ip neigh 

명령어로 리눅스 os 의 ARP 테이블에 접근할 수 있다.


![neigh table](https://github.com/user-attachments/assets/c39c08d5-589f-46c8-829b-831958fc60b3)


ARP 테이블은 [ip주소][연결 장치명][lladdr(L2주소) MAC addr][상태] 로 구분되어 표기된다.

NUD 상태의 의미는 다음과 같다.


1. INCOMPLETE : 목적지의 MAC 주소를 알기 위해 ARP 요청을 보낸 상태 (응답 대기 중)

2. REACHABLE : 최근 상위 계층 통신을 통해 해당 이웃이 정상적으로 도달 가능함이 확인된 상태

3. STALE : MAC 주소는 알고 있으나, 일정 시간 동안 사용되지 않아 도달 가능 여부가 검증되지 않은 상태

4️. DELAY : STALE 상태에서 패킷 전송 시, 즉시 ARP를 수행하지 않고 일정 시간 동안 상위 계층 응답을 기다리는 상태

5. PROBE : DELAY 상태에서 응답이 없을 경우, ARP 요청을 통해 도달 가능 여부를 재확인하는 상태

6. FAILED : ARP 요청에도 응답이 없어 해당 이웃이 도달 불가능하다고 판단된 상태


이 ARP 테이블은 MAC의 유효성 상태를 시간에 따라 변화시킨다.

  INCOMPLETE → REACHABLE → STALE → DELAY → PROBE → REACHABLE
                                                ↘ FAILED

### 4. delay 상태 의심

여기서 위의 ARP 과정에 따른 테이블 변화를 보면, 

b가 a의 ARP Request 패킷을 받았을 때 a의 MAC 주소가 DELAY 상태로 테이블에 업로드됨을 확인할 수 있었다.

DELAY 는 즉시 ARP를 수행하지 않고 일정 시간 상위 계층 응답을 기다리는 상태이다.

그 기다리는 시간은 어느 정도일까?


![delay_first_probe](https://github.com/user-attachments/assets/a1b13703-c7dd-4d28-93c6-df780bebbefb)


  /proc/sys/net/ipv4/neigh/ens224/delay_first_probe

를 확인하면 5초임을 알 수 있다!

즉 b가 delay 상태에서 상위 응답을 받지 못했기에 PROBE 상태로 ARP req를 시도했을 수 있다.


--> ! 오류


b가 기다리는 상위 계층 응답에는 ICMP ping 이 포함된다. 

그러므로 b는 delay 5초 이내에 충분한 응답을 받았고 PROBE 상태로 전환될 이유가 없음.



### 5. REACHABLE -> aging 으로 STALE?

  /proc/sys/net/ipv4/neigh/ens224/base_reachable_time

을 확인하면 REACHABLE 상태에서 STALE 상태로 전환되기까지의 시간을 알 수 있다.

찾아보니 30초. 

aging 으로 인한 ARP request 전송은 아니다.


### 6. NUD 검증체계와 Positive confirmation

결국 약 5초 후 ARP request 가 발송되었다는 점과 발송시에 이미 target MAC addr를 알고있었다는 점을 고려,

"ARP request를 받고 reply 했음에도 ARP 캐시의 상태가 REACHABLE 이 아니었다"

는 결론에 도달.

여기에서 능동 검증(active confirmation), positive confirmation 개념이 등장.

"내가 검증하지 않은 응답은 신뢰하지 않는다"

사실 b는 a의 ARP request 와 ICMP echo request 에 응답만 했을 뿐, a에게 응답이 필요한 요청을 보낸 적 없음.

REACHABLE 전이 조건 : 이웃에게서 "positive reachability confirmation" 받아야 함.

그 조건은, 내가 이웃에게 보낸 트래픽에 대해 이웃이 내게 unicast 로 응답해야 함.

즉 내가 직접 ICMP,TCP,ARP PROBE 등을 보내고 이웃이 응답을 unicast 로 반환해 줘야 REACHABLE 상태로 전이.

b는 reply 만 했기 때문에 능동 검증이 없음. 

### 결론

그러므로 b의 arp 테이블에서 a의 상태는 REACHABLE 이 아니었을 것이다.

STALE 상태에서 응답 전송이 발생하면서, 커널이 DELAY 상태로 들어갔지만 

positive confirmation 조건을 달성하지 못했음. (이웃의 reply가 아닌 request만 받음)

그래서 5초간 기다린 후 PROBE -> a에 대한 ARP Request를 발생시킨 것.



### 느낀 점

난 이 로그를 보고 단순한 GNS3 의 오류라고 생각했는데,

파고들어 보니 arp 테이블의 상태 전이 원리가 녹아 있는 메세지였음.

깊게 생각하지 않으려는 태도를 버리고 탐구하는 자세를 가져야겠다.
















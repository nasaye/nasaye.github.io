---
title : wireshark 패킷 간단 분석
date : 2026-04-21 01:45:00 +0900
category : study
tag : network, study
---

### ARP Announcement

  252	148.702406	VMware_03:d5:3c	Broadcast	ARP	42	ARP Announcement for 172.16.16.97

이게 Gratuitous ARP.

자기 IP 를 알리는 ARP. 인터페이스 up 때 또는 ip 설정 바뀔 때, ip 충돌확인 할 때

ARP 테이블 갱신하라고 브로드캐스트 하는 패킷. 

OPcode 1, [Is gratuitous: True], [Is announcement: True],

Destination: Broadcast (ff:ff:ff:ff:ff:ff)

Sender IP address: 172.16.16.97

Target MAC address: 00:00:00_00:00:00 (00:00:00:00:00:00)

Target IP address: 172.16.16.97



자기 자신 IP 한테 보내는게 자기 알리려고. 

브로드캐스트니까 특정 대상 없음.







### STP BPDU

스위치가 네트워크 루프를 방지하기 위해 루트를 기준으로 트리 구조를 만듬



IEEE 802.1D 

루트 브릿지 선출 -> 경로비용 계산 -> 포트의 역할 결정 -> 포트의 상태 변화  --> 를 계속 갱신


Destination: Nearest-Customer-Bridge : 제어 프레임



  2	0.586308	c4:01:27:dc:f1:01	Nearest-Customer-Bridge	STP	60	Conf. TC + Root = 32768/0/c4:01:0a:b8:00:00  Cost = 19  Port = 0x802a



Conf → Configuration BPDU (토폴로지 정보 전달)

TC → Topology Change 발생 또는 플래그 (포트 업다운, 링크, 장비 연결)

Root = 32768/... → 현재 루트 브리지 정보 (Priority / VLAN / MAC) : 

Cost = 19 → 루트까지 거리 : 1홉

Port = 0x802a → 송신 포트 ID 



이건 TC 발생으로 STP 변화한 것. 네트워크 장비들끼리 2초정도마다 갱신.


### IPv6 Router Solicitaion



  9	10.892105	fe80::20c:29ff:fe03:d53c	ff02::2	ICMPv6	62	Router Solicitation



  Source      fe80::20c:29ff:fe03:d53c
  
  Destination ff02::2 // 같은 링크의 모든 라우터에게 멀티캐스트
  
  ICMPv6      Router Solicitation


같은 링크에 있는 IPv6 라우터에게 라우터 광고 보내라는 패킷.

호스트가 네트워크에 붙을때 자동 발송 -> 기본 게이트웨이 , 프리픽스, ipv6 네트워크 존재확인용



즉 이거 뜨면 새 호스트가 토폴로지에 들어왔다는 뜻



### IPv6 Neighbor Solicitaion


  256	149.506284	::	ff02::1:ff03:d53c	ICMPv6	86	Neighbor Solicitation for fe80::20c:29ff:fe03:d53c



나는 아직 이 IPv6 주소 확정 아님

DAD : Duplicate Address Detection . 중복 주소 확인. 지가 쓰려는 주소가 타겟에 있음





### CDP



  22	36.127021	c4:01:27:dc:f1:01	CDP/VTP/DTP/PAgP/UDLD	CDP	343	Device ID: ESW1  Port ID: FastEthernet1/1  



Cisco 장비끼리 뿌리는 광고 패킷. Device id 가 패킷보낸 장비, Port ID 가 패킷 나온 포트.

그냥 내가 ESW1 스위치고 F1/1 포트에 연결됨 광고.

장비이름/IOS버전/인터페이스/플랫폼/duplex 상태 멀티캐스트 전송.

udp 로 뿌리는데 사이즈가 커서 그런지 스트림 인덱스가 1 이다.


### DHCP Discover



  72	131.426584	0.0.0.0	255.255.255.255	DHCP	342	DHCP Discover - Transaction ID 0x3fa71017



src IP 0.0.0.0 255.255.255.255 를 보면 알 수 있듯

클라이언트가 DHCP 설정으로 IP 주소를 받으려고 브로드캐스팅으로 DHCP 서버에 요청중.



Discover(클라->브로드) -> Offer (서버->클라)-> Request(클라->서버) -> ACK(서버->확정) 순.



이게 떴다는건 누가 수동IP설정 안하고 DHCP 설정 했다는건데

MAC c0:00:13 . 그런데 이런 MAC 를 가진 호스트가 없다.

그렇다면 숨겨진 장치가 있는 것일까?



이 로그의 src mac이 Source: VMware_c0:00:13 (00:50:56:c0:00:13) 이다.

즉 VMware  가상 네트워크쪽 인터페이스일 가능성이 높음.





### LLDP 

Link Layer Discovery Protocol



장비가 자기 정보를 이웃에게 알리는 표준 L2 탐색 프로토콜



  89	151.970589	VMware_c0:00:13	Nearest-Bridge	LLDP	67	LA/DESKTOP-3TCKI40 MA/00:50:56:c0:00:13 3601 



이거 방금 위 DHCP 요청한 애랑  MAC 주소가 똑같다.



시스템 이름 : DESKTOP-3TCKI40. 



이건 가상 네트워크에 연결된 vmnet 의 nic 에서 발생시킨 신호였다.



   Description . . . . . . . . . . . : VMware Virtual Ethernet Adapter for VMnet19
   
   Physical Address. . . . . . . . . : 00-50-56-C0-00-13

   DHCP Enabled. . . . . . . . . . . : No



그런데 DHCP Enabled 가 no 인데 왜..?



DHCP로 IP를 할당받지 않았지만 DHCP 요청은 할수도 있음.

DHCP 서버가 토폴로지상에 존재하지 않아서 

가상 인터페이스가 IP를 얻기 위해 Discover 패킷을 반복적으로 전송중인 상태.




### LLMNR

ipv4,ipv6


  206	222.961126	fe80::dfa5:fdeb:e2aa:bc3d	ff02::1:3	LLMNR	95	Standard query 0xbab2 ANY DESKTOP-3TCKI40
  
  207	222.961160	192.168.106.1	224.0.0.252	LLMNR	75	Standard query 0xbab2 ANY DESKTOP-3TCKI40



l2 네트워크에 이 이름 가진 장치 직접 물어보는 프로토콜. 브로드/멀티캐스트.

L2 한정. 이름으로 IP(로컬) 찾음. 우리동네 DNS.

DHCP 실패해서 이름 해석 필요하니까 발생



![범인](https://github.com/user-attachments/assets/59aaf0b3-c546-4ceb-9825-19f65aaaa088)

원인 찾았다. 이 항목에 체크해놔서 host 가상 어댑터가 네트워크에 들어와서 IP 달라고 날뛰고 있던 것.

내일 실습할 때 꺼보자.


### MLDv2 Report


  259	150.530261	fe80::20c:29ff:fe03:d53c	ff02::16	ICMPv6	90	Multicast Listener Report Message v2


인터페이스 IPv6 활성되면 자동 발생

DAD 이후~ 어떤 멀티캐스트 그룹 들을지 등록

IGMP 와 같은 역할.

03:d53c 는 4.4.4.1 IP.

4.4.4.1 에 IPv6 설정이 켜져있다는 뜻


![3c](https://github.com/user-attachments/assets/f33ae625-cb29-4e0e-a8e7-b95004c38c0b)

DAD 하면서 갖고싶은 IPv6 주소 광고하더니 결국 가져가는 모습이다

IP주소 얻었다고 바로 라우터에게 광고 보내라고 함.


### BROWSER

  281	164.730297	192.168.139.1	192.168.139.255	BROWSER	228	Request Announcement DESKTOP-057DE6Q

NetBIOS Browser Announcemnet

브로드캐스트로 윈도우 네트워크 탐색. 

윈도우가 주기적으로 이름등록, 네트워크에 광고 함.

지금 내 vmnet 19 (내 윈도우의 가상NIC) 인터페이스가 존재해서 Host 윈도우가 방송중.



## 최종 프로콜 목록

 L2 (데이터링크)

Ethernet

ARP

STP

LLDP

CDP


 L3 (네트워크)

IPv4

IPv6

 L3 제어 / 보조

ICMP

ICMPv6 (NDP, RS, NS 등)

 IP 설정 / 주소

DHCP

 이름 해석

DNS

LLMNR

NetBIOS

 멀티캐스트 관리

IGMP

MLD









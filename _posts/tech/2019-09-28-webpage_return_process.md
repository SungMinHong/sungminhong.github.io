---
title: "[웹 통신] 웹페이지 반환 과정"
date: 2019-09-28 17:04:28
lastmod : 2019-10-06 12:00:00
categories: network
tag: [web, network]
sitemap :
  changefreq : daily
  priority : 1.0
---

# 브라우저 주소창에 도메인을 입력하고 웹 페이지를 받는 과정.

&nbsp; 우선 웹브라우저에서 도메인을 입력해보자. 물론 도메인 없이 ip로도 웹페이지 접근이 가능하다.
그 이후 과정을 4단계로 나눠 설명하면 아래와 같다.

## 1. 브라우저의 캐시메모리와 PC의 hosts파일 확인
&nbsp; 브라우저는 브라우저 혹은 운영체제의 캐시메모리에서 해당하는 도메인 주소가 있는지 확인한다. 있는 경우 DNS서버를 사용하지 않는다. 없는 경우 로컬PC의 hosts파일을 읽어 입력한 도메인의 매핑정보가 존재하는지 확인한다. 이 파일은 PC 내부의 DNS 역할을 한다. 1단계에서 IP주소를 찾는다면 추가적으로 DNS서버를 사용하지 않고 IP를 이용해 타겟 서버로 통신할 수 있다. 
<br/> 

## 2. DHCP & ARP
 &nbsp; DHCP는 Dynamic Host Configuration Protocol의 약자로, 호스트의 IP주소 및 TCP/IP 설정을 클라이언트에 자동으로 제공하는 프로토콜이다.
 사용자의 PC는 DHCP 서버에서 사용자 자신의 IP주소, 가장 가까운 라우터의 IP주소, 가장 가까운 DNS 서버의 IP주소를 받는다.
 이후 ARP 프로토콜을 이용하여 IP주소를 기반으로 가장 가까운 라우터의 MAC주소를 알아낸다.

![DHCP Server의 동작 개념도](https://t1.daumcdn.net/cfile/tistory/267BCC405870914920)
<br/> 

## 3. IP 정보 수신
&nbsp; 두 번째 과정을 통해 외부와 통신할 준비를 마쳤으므로, DNS Query를 DNS 서버에 송신한다.
DNS 서버는 이에 대한 결과로, 웹 서버의 IP 주소를 사용자 PC에 돌려준다.
DNS 서버가 도메인에 대한 IP 주소를 송신하는 과정은 약간 복잡하다.
사용자의 PC는 가장 먼저, 지정된 DNS서버(우리나라의 경우 통신사 별로 지정된 DNS서버가 있다.)에 DNS Query를 송신한다.
(http://www.naver.com으로 가정하자) 그 후 지정된 DNS서버(인터넷 서비스 제공자가 제공한)는 요청 도메인에 해당하는 IP주소가 캐시되어 있는지 확인한다. 만약 없으면 Root 네임서버에 www.naver.com을 질의하고, Root 네임서버는 .com 네임서버의 ip주소를 알려준다. 
 그 후 .com 네임서버에 www.naver.com을 질의하면 naver.com 네임서버의 ip주소를 받고, 그곳에 질의를 또 송신하면 www.naver.com의 IP주소를 수신하게 된다.
이와 같이 여러번 왔다갔다 하는 이유는, 도메인의 계층화 구조에 따라 DNS서버도 계층화되어있기 때문이다. 이렇게 계층화되어 있으므로 도메인의 가장 최상단, 즉 가장 뒷쪽(.com, .kr 등등)을 담당하는 DNS서버는 전세계에 13개 뿐이다.
![야후 도메인 찾는 과정](https://user-images.githubusercontent.com/18229419/61994113-58fa2800-b0b1-11e9-98de-d409bcc4cbff.png)
<br/>
위 그림의 Resolver서버는 DNS서버와 같다. DNS서버가 IP주소를 찾기위해 각 서버들을 재귀적으로 호출한다 하여 재귀적 Resolver 서버라고도 한다. [관련링크](https://serverfault.com/questions/422288/what-is-the-difference-between-authoritative-nameserver-and-recursive-resolver)

> 좀 더 자세한 IP찾는 과정을 원한다면 -> [링크로 이동](https://www.youtube.com/watch?v=mpQZVYPuDGU)


## 4. 웹 서버 접속
&nbsp; 브라우저는 http 통신을 통해서 사이트 문서를 가져오고 이를 해석해 화면에 출력하게 된다. http 요청을 하기 위해 TCP Socket을 개방하고 연결한다. 이 과정에서 서버와 3-Hand-Shaking이 일어난다.TCP 연결에 성공하면, Http Request가 TCP Socket을 통해 보내진다. 이에 대한 응답으로, 웹 페이지의 정보가 사용자의 PC로 들어온다. 만약 HTML을 필요로하는 요청이었다면 이후 렌더링 엔진이 HTML과 CSS를 파싱하여 화면에 표시한다. 
<br/> 
> 좀 더 자세한 랜더링 과정을 원한다면 -> [링크로 이동](https://d2.naver.com/helloworld/59361)
<br/> 
&nbsp; 통신하는 과정을 좀 더 자세히 정리하면 다음과 같다. TCP/IP 계층의 순서로 네트워크에 접근한다. HTTP 프로토콜은 Application계층(4계층)에 속해 있다. Application계층에서는 정보를 만들어 전달한다. Transport계층(3계층)에서는 통신 노드를 연결한다. Internet계층(2계층)에서는 통신노드간 패킷을 전송하고 라우팅한다. Network Access계층(1계층)에서 전기적 신호로 변환하여 실제로 전달한다. 수신 측에서는 반대로 낮은 계층에서 윗 계층으로 올라가며 해석된다.
<br/> 

## 자세한 용어 설명
### HTTP
&nbsp; HTTP는 Hypertext Transfer Protocol의 약자로 인터넷상에서 데이터를 주고받기 위한 프로토콜이다. 데이터는 오디오 / 비디오 / 이미지 / 텍스트 등 어떠한 데이터의 종류를 가리지 않고 모두 HTTP 프로토콜을 이용해 전달하고 전달 받을 수 있다.
<br/> 
### 네트워크 계층
- OSI 7계층, TCP/IP 4계층
![](https://t1.daumcdn.net/cfile/tistory/213F623C566BAE253B)
<br/>

1계층 네트워크 액세스 계층(Network Access Layer or Network Interface Layer)
- OSI 7계층의 물리계층과 데이터 링크 계층에 해당한다.
- 물리적인 주소로 MAC을 사용한다.
- LAN, 패킷망, 등에 사용된다.

2계층 인터넷 계층(Internet Layer)
- OSI 7계층의 네트워크 계층에 해당한다. 
- 통신 노드 간의 IP패킷을 전송하는 기능과 라우팅 기능을 담당한다.
- 프로토콜 – IP, ARP, RARP

3계층 전송 계층(Transport Layer)
- OSI 7계층의 전송 계층에 해당한다.
- 통신 노드 간의 연결을 제어하고, 신뢰성 있는 데이터 전송을 담당한다.
- 프로토콜 – TCP, UDP

4계층 응용 계층(Application Layer)
- OSI 7계층의 세션 계층, 표현 계층, 응용 계층에 해당한다.
- TCP/UDP 기반의 응용 프로그램을 구현할 때 사용한다.
- 프로토콜 - FTP, HTTP, SSH

<br/> 

> 출처: <br/>
[How the DNS works](https://www.youtube.com/watch?v=2ZUxoi7YNgs#action=share) <br/>
[medium 박성룡: Website는 어떻게 보여지게되는 걸까?](https://medium.com/@pks2974/website%EB%8A%94-%EC%96%B4%EB%96%BB%EA%B2%8C-%EB%B3%B4%EC%97%AC%EC%A7%80%EA%B2%8C%EB%90%98%EB%8A%94-%EA%B1%B8%EA%B9%8C-1-108009d4bdb) <br/>
[TCP/IP 4계층(TCP/IP 4 Layer)](https://hahahoho5915.tistory.com/15)

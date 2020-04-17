---
title: "[Http Client] OkHttp 동작 흐름도 정리"
date: 2020-04-12 16:00:00
lastmod : 2020-04-12 16:00:00
categories: [Http]
tags: [Http, client, study]
sitemap :
  changefreq : daily
  priority : 1.0
---
한달 전쯤 HTTP 관련 [Nagle's algorithm 발표](https://github.com/Study-Java-Together/study-http/blob/master/documents/member/sungminhong/what-okHttp.md)를 진행할 때 받았던 질문에 대해 조사해봤습니다.
그 때 한 분이 TCP slow start와 MTU의 관련성을 질문해주셨고 이를 조사해 본 내용을 공유 드리고자 합니다. 좋은 질문 주셔서 감사합니다 :heart_eyes:

~~~
질문: TCP slow start와 MTU(maximum transmission unit: 최대 전송 단위)는 관련이 있을까요?
~~~

알아 본 결과 결론은 아래와 같습니다. :grin:
~~~
답변: MTU와는 관련이 있습니다. 하지만 Nagle's algorithm과는 관련이 없습니다.
~~~
그렇게 생각한 이유를 아래와 같이 정리해봤습니다. :lying_face:

## 1. TCP slow start
TCP slow start는 window size를 점차 늘리면서 통신하는 방법입니다. 이는 송신측의 전송 속도가 수신측의 수신 속도보다 빨라 발생하는 문제를 해결하기 위해 고안된 알고리즘입니다.
가장 초기에 설정하는 window 크기는 다음과 같은 명칭을 가지고 있습니다. initial congestion window size(CWND).
그리고 window size 증가는 다음과 같이 진행될 수 있습니다.
~~~
CWND * 1-> CWND * 2 -> CWND * 4 -> CWND * 8    //지수 증가
~~~
+) 최대로 증가할 수 있는 window size는 TCP header의 window size field를 확인해야 알 수 있습니다. 추가 설정을 하지 않으면 2*16 = 65535byte = 64kbyte가  전송 가능한 최대 window size입니다. TCP header, RFC 1323에서는 TCP window scaling이라는 옵션을 정의하고 있습니다. TCP 헤더의 옵션 필드에 window scale라는 필드를 정의하여 advertise할 수 있는 receiver window size를 키울 수 있습니다.

## 2. TCP slow start와 MTU는 어떤 관계가 있을까요?
TCP slow의 첫 송신(위 숫자 중 1번에 해당)에 사용되는 window size인 CWND가 MTU와 관련이 있습니다. 첫 송신에 보낼 윈도우 크기를 MSS(maximum segment size)에 따라 결정됩니다. 그리고 MSS는 다음과 같이 계산될 수 있습니다.
~~~
MSS = MTU - TCPHdrLen - IPHdrLen
~~~
IPv4기준으로 계산한다면 아래와 같이 1460byte입니다.
~~~
MSS = 1500 - 20 - 20 = 1460(byte)
~~~
+) Ethernet, MTU, MSS 크기
![Ethernet, MTU, MSS 크기](https://www.networkcomputing.com/sites/default/files/MSS-2.png)
위 기준을 단순화해 CWND = MSS로 가정하고 계산해본다면 클라이언트는 ACK를 받을 때 마다 아래와 같이 window 사이즈를 증가 시킵니다. 10번의 ACK가 성공적이었다면 아주 큰 데이터도 이렇게 보낼 수 있다고 합니다. (CWND를 결정하는 조건은 [RFC5691](https://tools.ietf.org/html/rfc5681)를 확인해주세요. 보통 TCP / IP 스택 구현에 따라 다르다고 합니다. 결국 이 쪽 설정은 시스템엔지니어링 분야인 것 같습니다.. ㅎ 참조:  [리눅스 서버 커널 파리미터 이야기](https://meetup.toast.com/posts/53) )
~~~
1460 * 1 -> 1460 * 2 -> 1460 * 4 -> 1460 * 8 -> ... -> 1460 * 1024 
// TCP header내 window size field 값보다 작고 수신자 측의 최대 수용 가능한 Window Size보다 작은 경우 증가할 수 있습니다.
~~~
위 방법처럼 한번에 여러 segment를 한번에 보내는 방법을 [sliding window](https://ko.wikipedia.org/wiki/%EC%8A%AC%EB%9D%BC%EC%9D%B4%EB%94%A9_%EC%9C%88%EB%8F%84)라고 합니다. 반면에 1 개를 보내고 ACK를 수신한 이후에 다시 보내는 방법은 stop and wait 방식입니다. 오늘날은 전송 효율을 올리기 위해 sliding window를 사용하고 있습니다.

## 3. 결론
제가 낸 결론은 다음과 같습니다. :man_dancing:
1. MTU와 TCP Slow Start는 관련이 있습니다.
MTU를 통해 MSS가 결정되고 MSS에 의해 CWND가 결정되고 CWND가 TCP slow start에서 사용됨으로 둘은 간접적으로 관련이 있습니다.
2. Nagle's algorithm과 TCP Slow Start는 관련이 없습니다.
Nagle’s algorithm에서도 MTU를 이용해 버퍼의 크기를 결정합니다. MTU를 사용한다는 공통점만 있을 뿐 둘의 연관성은 없습니다. Nagle’s algorithm 구현에 사용되는 버퍼의 크기가 의미하는 바는 언제까지 데이터를 보내지 않고 기다릴 것 인지를 의미합니다. 즉 얼마나 큰 데이터를 보낼지와는 관련이 없습니다. 만약 크기가 너무 큰 경우 버퍼에 쌓지 않고 write를 할 뿐입니다(여기까지만 애플리케이션에서 통제). 그 이후 TCP slow start에 의해 정해진 최대 전송 가능 window size에 따라 segment가 전송됩니다.

## TMI :zipper_mouth_face:
아래 그림과 같이 네트워크 망에 문제가 발생했을 때는 도표와 같이 window size를 전체적으로 낮추고 네트워크 망이 회복되면 Fast Retransmit이라는 방법으로 window size를 빠르게 복구한다고 합니다.
![Fast Retransmit](https://user-images.githubusercontent.com/18229419/79529495-c3238480-80a7-11ea-929e-c0910db20402.png)

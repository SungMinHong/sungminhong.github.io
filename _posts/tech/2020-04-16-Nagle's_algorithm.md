---
title: "[HTTP] HTTP Client에는 Nagle's_algorithm이 있다?"
date: 2020-04-16 16:00:00
lastmod : 2020-04-16 16:00:00
categories: [Http]
tags: [HTTP, MTU, Nagle's algorithm]
sitemap :
  changefreq : daily
  priority : 1.0
---
오픈소스인 HTTP Client중 하나인 ok-http1.0 을 디버깅 하며 Soket에 직접 write하는 시점이 언제 일지를 찾아봤는데요.
여러 삽집을 하며 [Nagle](https://en.wikipedia.org/wiki/Nagle%27s_algorithm)이라는 알고리즘을 발견했습니다! :sparkles: 
이번 포스트에서는 디버깅하며 발견한 Nagle's_algorithm에 대해 공유드리겠습니다. :wink:

### 1. Soket에 데이터를 쓰는 시점은 언제일까요?
- socket write가 실행되는 지점이 어디인지 찾기 위해 ok-http 디버깅을 해봤습니다. (쉽지 않습니다. 타임아웃이 계속 발생하기 때문에..)
- 웹 클라이언트 실제 구현을 확인해 보면 [네이글 알고리즘](https://en.wikipedia.org/wiki/Nagle%27s_algorithm)이 사용된다는 사실을 알 수 있습니다.
- 결론적으로 네이글 알고리즘으로 인해 데이터를 쓰는 시점은 다양할 수 있습니다.

### 2. [네이글 알고리즘](https://ozt88.tistory.com/18)이란?
IP 네트워크에서 데이터는 몇 곂의 헤더로 캡슐화되어 목적지로 보내집니다. 이 헤더들의 용량도 제법 커서, 적은 데이터를 보내게되면 배보다 배꼽이 커지는 경우가 발생합니다. 고의로 작은 단위의 데이터를 전송하는 경우도 있겠지만, 의도치 않게 네트워크 상황상 비효율적인 송신을 해야하는 경우가 있습니다. 
<br/>
예를 들면 전송해야될 데이터가 있는데, 상대방의 윈도우 크기(전송 받을 수 있는 크기)가 매우 작은 경우. 의도한 바는 아니지만 보낼 수 있는 패킷의 크기 자체가 작기 때문에 따로 지연 설정을 하지 않으면, 작은 크기의 패킷이 만들어질 수 밖에 없습니다.
<br/>
보낼 수 있는 데이터를 바로 패킷으로 만들지 않고, 가능한 버퍼에 모아서 더 큰 패킷으로 만들어 한번에 보내면 이런 문제는 발생하지 않을 것입니다. 네이글 알고리즘은 이 대안을 실제로 구현한 네트워크 전송 알고리즘입니다.
<br/> 
아래는 네이글이 On, Off 됨에 따라 네트워크 통신이 이뤄지는 모습을 보여줍니다. On인 경우 더 많은 데이터를 버퍼에 모았다가 한번에 보내는 것을 알 수 있습니다.
<br/>
![네이글 on/off](https://t1.daumcdn.net/cfile/tistory/232E58465514FBDE3D)
<br/>
결국 버퍼에 얼마나 쌓였냐에 따라 데이터를 실제로 socket에 write하는 시점이 바뀔 수 있습니다.

### 3. 그렇다면 네이글 알고리즘이 왜 적용됐던 거죠?
ok-http에서는 소켓을 생성할 경우 추가 적으로 setNoDealy를 통한 설정을 하지 않으므로 네이글 알고리즘이 동작합니다. 실시간성이 중요한 통신을 할 경우 이 옵션을 true로 설정하면 됩니다. Ex: FPS 게임
- 아래와 같이 socket 객체를 setNoDealy 메서드를 호출해 초기화 않았다면 네이글 알고리즘이 디폴트로 적용되게 됩니다. setNoDealy가 설정하는 SocketOptions는 [TCP_NODELAY](https://docs.oracle.com/javase/8/docs/api/java/net/SocketOptions.html#TCP_NODELAY)입니다.

~~~java
socket.setNoDelay(true);
~~~

ok-http에서는 native메서드를 통해 받아온 [MTU](https://ko.wikipedia.org/wiki/%EC%B5%9C%EB%8C%80_%EC%A0%84%EC%86%A1_%EB%8B%A8%EC%9C%84)(maximum transmission unit: 최대 전송 단위) 값으로 버퍼 사이즈를 설정하게 됩니다. (버퍼를 왜 가지고 있는지 궁금하시다면 꼭 네이글알고리즘을 확인해주세요)

### 4. 자. 이제 이렇게 테스트 해봐요 (ok-http1.0 기준)
먼저 직접 테스트를 하기 위해 output을 보내도록 해야합니다. 아래 예시와 같이 설정하면 됩니다. POST 메서드로 요청할 서버로 tcpschool을 사용했습니다. ok-http 1.0 기준으로 작성됐습니다.

~~~java
        String param = "city=Seoul&zipcode=06141";
        HttpURLConnection connection = client.open(new URL("http://tcpschool.com/examples/media/request_ajax.php"));
        connection.setRequestMethod("POST");
        connection.setDoOutput(true);
        connection.setFixedLengthStreamingMode(param.getBytes().length);  //이렇게 fixed 해줘도 되고 안 해줘도 괜찮습니다.
        OutputStream outputStream = connection.getOutputStream();         //여기서 outputStream을 가져오며 소켓 생성과 연결이 이루어졌습니다.
        outputStream.write(param.getBytes()); //여기서 write 요청을 하고 네이글 알고리즘에 맞게 동작합니다.
~~~

### 5. 어떻게 동작할까요: MTU, Header, Body 값에 따른 동작
1. MTU가 100인 경우
    1. 보내는 header, body 데이터를 다 합쳤는데 byte 길이가 99인 경우
        1. 데이터를 다 넣었는데 버퍼 사이즈를 넘기지 않았습니다. 계속 넣어두고 기다립니다.
        2. readResponse() 내부 로직에서 flush 하는 로직이 있습니다. 여기서 socketInputStream으로 보내게 됩니다.
    2. 보내는 header는 100이고 body가 100인 경우
        1. header 데이터를 socketInputStream으로 보내게 됩니다.
        2. 이후 body 데이터를 write하는 요청을 하는 경우 socketInputStream으로 보내게 됩니다.
        3. readResponse() 때 flush를 하려고 해도 남아 있는 데이터가 없습니다.
    3. 보내는 header는 99이고 body가 100인 경우
        1. 버퍼 크기를 넘지 않았으니 header 데이터를 버퍼에 넣어두고 기다립니다.
        2. 바디 write 요청을 하는 경우 버퍼크기를 넘기면서 header를 먼저 socketInputStream으로 보내고 body 데이터를 바로 이어서 보냅니다.
        3. readResponse() 때 flush를 하려고 해도 남아 있는 데이터가 없습니다.


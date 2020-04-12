---
title: "[Nginx] Nginx 개선을 위한 사전 조사"
date: 2020-04-09 16:00:00
lastmod : 2020-04-12 16:00:00
categories: [survey]
tags: [nginx, porxy, cache-contorl, customizing, survey]
sitemap :
  changefreq : daily
  priority : 1.0
---

최근에 Nginx에서 cache-control 헤더를 좀 더 개선('no-cache'에서 'no-store'로 변경)할 수 있을지 조사를 해봤습니다.

## Nginx에서 cache-control 헤더 관련  조사
결론적으로 프록시(ex: Nginx, Apach 등)에서  cache-control 헤더를 'no-cache'에서 'no-store'로 변경 하더라도 차이가 없을 것 같습니다.
cache-control 헤더는 클라이언트(request)와 서버(response) 모두에서 사용이 가능했습니다. 하지만 어디에서 사용하는 지에 따라 의미가 살짝 다른 면이 있었습니다. (참조: [request, response내 cachce-control header 의미 차이](http://shorturl.at/cwIN3))
request의 cache-control 헤더에 no-cache attribute가 담겨 있다면 이 의미는 "end-to-end reload" 와 같은 의미였습니다.
현재 Nginx에 있는 config 설정은 아래와 같이 설정돼 있었습니다.

~~~
proxy_set_header Cache-Control "no-cache";
~~~

그렇기 때문에 Nginx도 proxy가 되어 (마치 클라이언트와 같이)request의 cache-control 헤더에 no-cache를 담은 것이고 이는 "다음 프록시 혹은 서버에게 캐시를 사용하지 말고 실제 데이터를 줘"라는 의미로 사용될 수 있었습니다.
의도하는 대로 잘 동작하고 있고 request cache-control 헤더에 no-store를 사용해야 한다는 자료를 찾아보지 못했습니다. 논리적으로 no-cache와 동일하기 때문인 것 같습니다.

단순히 잘 돌아가니 놔두자라는 생각을 버리고 개선점이 있는지 꾸준히 탐구하고자 합니다. 시간이 날 때마다 추가로 조사해보려고 합니다.

---
출처: <br/>
[why-is-cache-control-attribute-sent-in-request-header-client-to-server](https://stackoverflow.com/questions/14541077/why-is-cache-control-attribute-sent-in-request-header-client-to-server) <br/>
[why-use-cache-control-header-in-request](https://stackoverflow.com/questions/42652931/why-use-cache-control-header-in-request) <br/>
[whats-the-difference-between-cache-control-max-age-0-and-no-cache](https://stackoverflow.com/questions/1046966/whats-the-difference-between-cache-control-max-age-0-and-no-cache) <br/>

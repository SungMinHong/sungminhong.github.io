---
title: "[스프링 스탑워치] "
date: 2019-09-29 20:23:28 -0400
categories: 스프링 시큐리티
---

# Spring StopWatch 테스트

- Spring StopWatch를 사용하지 않는 경우 개발자들은 대개 다음과 같은 방식으로 stop watch 기능을 구현한다. 
시스템(System) 클래스에서 밀리세컨드 현재 밀리세컨드 값을 가져와 그 차이를 구하는 방식을 사용한다.

~~~java
@Test
public void testSimpleTimer() {
    long startTime = System.currentTimeMillis();
    int data = 0;
    for(int i = 0; i < 1_000_000; i++) {
        data += 1;
    }
    long elapsedTime = System.currentTimeMillis() - startTime;
    System.out.println(elapsedTime);
}
~~~
<br/>
<br/>

- 다음 예제는 스프링 프레임워크에서 제공하는 유틸 중 하나인 org.springframework.util.StopWatch에 대한 간단한 테스트다.

~~~java
public class StopWatchTest {

    private StopWatch stopWatch;

    @Before
    public void setUp() {
        stopWatch = new StopWatch("honeymon");
    }

    /**
     * Long 타입과 BigDecimal 타입의 덧셈 소요시간 비교:
     */
    @Test
    public void testAddLongAndBigDecimal() {
        BigDecimal bigDecimal = BigDecimal.valueOf(0, 0);
        Long longType = 0L;

        stopWatch.start("Long type");
        for(int i = 0; i < 1_000_000; i++) {
            longType += 1L;
        }
        stopWatch.stop();

        stopWatch.start("BigDecimal type");
        for(int i = 0; i < 1_000_000; i++) {
            bigDecimal = bigDecimal.add(BigDecimal.ONE);
        }
        stopWatch.stop();
        
        System.out.println(stopWatch.prettyPrint());
    }
}
~~~
<br/>
<br/>
- 테스트 결과
다음과 같이 총 실행 시간 중 BigDecimal type 계산이 약 60프로를 차지하며 대략 1.5배 정도 더 느리다는 것을 확인할 후 있다.
<br/>

~~~
...
-----------------------------------------
ms % Task name 
----------------------------------------- 
00017 040% Long type
00025 060% BigDecimal type 
~~~
<br/>

+) 티몬의 베스트 탭 api 속도 개선을 위해 오래걸리는 로직을 찾기위해 StopWatch를 사용했다. 메서드 내에서 특정 반복문 등의 시간을 측정할 때 적합하다.

> 출처: https://java.ihoney.pe.kr/506 

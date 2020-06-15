---
title: "[Spring] 슈퍼타입토큰"
date: 2020-06-14 00:00:00
lastmod : 2020-06-14 00:00:00
categories: [English]
tags: [Spring, 자바, 타입토큰, 슈퍼타입토큰]
sitemap :
  changefreq : daily
  priority : 1.0
--- 
# 서론
 &nbsp; 혹시 슈퍼타입 토큰이라는 단어를 들어보셨나요? 저는 사용은 해봤지만 정확한 동작원리는 몰랐습니다. 😵
 저는 모른다는 것을 인식하고 알기 위해 노력하는 것이 개발자의 덕목이라고 생각합니다. 그래서 저는 슈퍼타입토큰이 무엇이고 어떻게 활용할 수 있는지 처음부터 쭉 조사하고 정리하려고 합니다. 🤗
 제가 단계적으로 궁금했던 점을 목차로 만들어 봤습니다. 😁 <br/>

## 목차 🤔
1. JAVA 제네릭(Generic)은 뭐죠?
2. JAVA 제네릭은 런타임 때 유효한가요?
    1. JAVA와 C#의 제네릭 비교
    2. 왜 JAVA는 Type Erasure를 채택했나요?
    3. Type Erasure
3. 타입토큰(Type Token) 이란?
4. 자바의 제네릭 구현 때문에 런타임때 타입토큰을 알 수 없다. 그러면 방법이 없나요?
    1. 슈퍼타입토큰(Super Type Token) 을 이용하면 해결할 수 있어요!
5. 하지만 구현이 너무 복잡하네요.. 더 쉬운 방법은 없을까요?
    1. 스프링에서 제공하는 슈퍼타입토큰을 사용해봅시다
6. Spring 슈퍼타입토큰은 주로 어디에서 활용 가능할까요?

# 본론 🧐
## 1. JAVA 제네릭(Generic)은 뭐죠?
- 클래스에서 사용할 타입을 외부에서 정하는 것을 의미합니다.
- 선언시 클래스 또는 인터페이스에 '< >' 기호를 쓰고 그안에 타입을 넣으면 됩니다.
- 제네릭을 사용함으로써 Object형을 강제 형변환할 필요가 줄어들었습니다.
~~~ java 
// 사용 예시
List<String> stringList = new ArrayList<>();
~~~
- 제네릭을 사용 유무에 따른 테스트
~~~ java
public class GenericListSample {
    public List<String> list = new ArrayList<>();

    public void addString(String str) {
        list.add(str);
    }

    public String getString(int index) {
        return list.get(index);
    }
}

public class NonGenericListSample {
    public List list = new ArrayList();

    public void addString(Object obj) {
        list.add(obj);
    }

    public String getString(int index) {
        return (String) list.get(index);
    }
}
~~~

~~~ java
//테스트 코드
class GenericListSampleTest {
    private GenericListSample list = new GenericListSample();

    @Test
    public void 리스트에_String_삽입하고_가져오기_테스트() {
        String mockStr = "문자열1";
        list.addString(mockStr);
        String str = list.getString(0);
        assertEquals(str, mockStr);
    }

    @Test
    public void 리스트에_Integer_삽입하고_가져오기_테스트() {
        Integer mockInt = 1;
//        list.addString(mockInt);   //컴파일 타임에 오류 감지
        String str = list.getString(0);
        assertEquals(str, mockInt);
    }
}


class NonGenericListSampleTest {
    private NonGenericListSample list = new NonGenericListSample();

    @Test
    public void 리스트에_String_삽입하고_가져오기_테스트() {
        String mockStr = "문자열1";
        list.addString(mockStr);
        String str = list.getString(0);
        assertEquals(str, mockStr);
    }

    @Test
    public void 리스트에_Integer_삽입하고_가져올때_ClassCastException_발생_테스트() {
        Integer mockInt = 1;
        list.addString(mockInt);   //컴파일 타임에 오류 감지 못함
        //아래 처럼 런타임때 ClassCastException이란 예외 발생!
        Assertions.assertThrows(ClassCastException.class, () -> {
            String obj = list.getString(0);
        }); 
    }
}
~~~

## 2. JAVA 제네릭은 완전할까요?
- 자바에 제네릭이 추가되면서 컴파일 시기때 타입 안정성을 얻을 수 있었습니다.
- 이 덕분에 IDE에서도 타입이 맞지 않으면 빨간줄로 경고 해줄 수 있습니다.
- 그렇다면 런타임 타입 안정성 또한 있을까요?
### 2_1. JAVA와 C#의 제네릭 비교
![java_vs_c#](https://user-images.githubusercontent.com/18229419/84587561-30128b00-ae5b-11ea-8572-883c06e8bae0.png) <br/>
출처: https://www.guru99.com/java-vs-c-sharp-key-difference.html
-  C#은 런타임 타입 안정성이 있습니다. 반면에 JAVA의 제네릭은 런타임 타입 안정성이 없습니다. 컴파일 과정에서 Type Erasure 과정을 통해 타입 파라미터를 전부 지웠기 때문입니다.
### 2_2. 왜 JAVA는 Type Erasure를 채택했나요?
제네릭을 도입할 당시 JAVA와 C#은 보급률에 큰 차이가 있었습니다. JAVA로 구현된 현업 프로젝트가 월등히 많았던 것입니다. 
JAVA는 하위 호완성을 지키는 언어로 많은 사랑을 받았고 이를 꼭 지키고자 했습니다. 
때문에 자바 진영은 컴파일 과정에서 Type Erasure 과정을 통해 타입 파라미터를 전부 지워주고 제네릭이 없던 하위 버전과 동일한 형태로 class 파일을 생성합니다.
반면 C#은 당시 보급률이 현저히 낮았기 때문에 하위 호완성을 포기하고 컴파일, 런타임 시기에 모두 완전한 제네릭을 적용했습니다.
### 2_3. Type Erasure
- 아래 그림과 같이 컴파일 과정에서 타입을 소거하고 Object로 만듭니다.
![type_erasure](https://image.slidesharecdn.com/matoringenerics-160622172122/95/tricky-java-generics-17-638.jpg?cb=1466616158)

## 3. 타입토큰(Type Token) 이란?
### 3_1. 토큰이란?
일반적으로 토큰은 버스 요금 등에 사용하기 위하여 회사에서 발행한 동전 모양의 주조물을 의미합니다. 전산 분야에서는 다음과 같이 정의하고 있습니다.
~~~
가장 낮은 단위로 어휘 항목들을 구분할 수 있는 분류 요소
~~~
![토큰이미지](https://user-images.githubusercontent.com/18229419/84657891-f0799b00-af4f-11ea-8c42-b6baab8c236a.png)
### 3_2. 타입토큰이란?
 타입 토큰(Type Token)은 쉽게 말해 "타입을 나타내는 최소한의 단위"입니다. 자바언어 개발자였던 Neal Gafter는 JAVA JDK5에 generics를 추가할 때 java.lang.Class 가 generic type이 되도록 변경했다고 합니다. 예를들어, String.class의 Type이 Class<String> 되도록 한 것입니다. 또한 이를 명칭하기 위해 [Gilad Bracha](http://bracha.org/Site/Home.html)라는 분이 타입 토큰이라는 용어를 만들어 줬다고 합니다. 😮
  


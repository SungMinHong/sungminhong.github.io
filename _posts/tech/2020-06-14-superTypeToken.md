---
title: "[Spring] 수퍼 타입 토큰"
date: 2020-06-14 00:00:00
lastmod : 2020-07-12 00:00:00
categories: [Spring]
tags: [Spring, 자바, 타입토큰, 수퍼타입토큰]
sitemap :
  changefreq : daily
  priority : 1.0
--- 

<br/>
<br/>

# 시작하며..  🚗
 &nbsp; 혹시 슈퍼타입 토큰이라는 단어를 들어보셨나요? 저는 사용은 해봤지만 정확한 동작원리는 몰랐습니다. 😵
 저는 모른다는 것을 인식하고 알기 위해 노력하는 것이 개발자의 덕목이라고 생각합니다. 그래서 저는 슈퍼타입토큰이 무엇이고 어떻게 활용할 수 있는지 처음부터 쭉 조사하고 정리하려고 합니다. 🤗
 제가 단계적으로 궁금했던 점을 목차로 만들어 봤습니다. 😁 <br/>

## 목차 🤔
1. JAVA 제네릭(Generic)은 뭐죠?
2. JAVA 제네릭은 런타임 때 유효한가요?
    1. JAVA와 C#의 제네릭 비교
    2. 왜 JAVA는 Type Erasure를 채택했나요?
    3. Type Erasure
3. 타입토큰(Type Token) 이란 무엇일까요?
	1. 토큰이란?
	2. 타입 토큰의 정의한다면?
	3. 클래스 리터럴과 타입 토큰의 의미는?
	4. 타입 토큰은 어디에 쓰이나요?
	5. 타입 토큰의 한계점은?
4. 수퍼 타입 토큰(Super Type Token)으로 한계를 극복해 보아요!
	1. 수퍼 타입 토큰이란?
	2. List 내 타입과 같은 중첩된 타입은 어떻게 구할 수 있나요?
	3. super type token을 이용한 TypeSafeMap 만들어 보아요!
5. 구현이 너무 복잡하네요.. 이미 잘 만들어진건 없을까요? 🥺
	1. Spring의 ParameterizedTypeReference를 사용해주세요! 👏
6. 그렇다면 수퍼 타입 토큰은 주로 어디에 사용할 수 있을까요? 🤔

<br/>

---
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
- 혹자는 자바의 제네릭이 반쪽짜리 제네릭이라고 하기도 합니다. 그 이유는 자바의 런타임 타입 안정성과 관련이 있습니다 😲

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


## 3. 타입토큰(Type Token) 이란 무엇일까요?
### 3_1. 토큰이란?
![토큰이미지](https://user-images.githubusercontent.com/18229419/84657891-f0799b00-af4f-11ea-8c42-b6baab8c236a.png)
<br/>
- 일반적으로 토큰은 버스 요금 등에 사용하기 위하여 회사에서 발행한 동전 모양의 주조물을 의미합니다. 
- 전산 분야에서는 다음과 같이 정의하고 있습니다.
~~~
가장 낮은 단위로 어휘 항목들을 구분할 수 있는 분류 요소 🤗
~~~

### 3_2. 타입 토큰을 정의한다면?
 자바언어 개발자였던 Neal Gafter는 JAVA JDK5에 generics를 추가할 때 java.lang.Class 가 generic type이 되도록 변경했다고 합니다. 예를들어, String.class의 Type이 Class&lt;String&gt; 되도록 한 것이라고 합니다. 또한 이를 명칭하기 위해 [Gilad Bracha](http://bracha.org/Site/Home.html)라는 분이 타입 토큰이라는 용어를 만들어 줬다고 합니다. 😮 토큰의 전산적 의미를 고려한다면 타입 토큰은 이런 뜻이라고 조심스레 유추해봅니다 ㅎㅎ
~~~
타입을 나타내는 최소한의 단위"
~~~

### 3_3. 클래스 리터럴과 타입 토큰의 의미는?
- 클래스 리터럴(Class Literal)은 String.class, Integer.class 등을 말합니다.
- String.class의 타입은 Class&lt;String&gt;, Integer.class의 타입은 Class&lt;Integer&gt;입니다.
- 타입 토큰(Type Token)은 타입을 나타내는 토큰입니다.
- 클래스 리터럴은 타입 토큰으로서 사용됩니다.
- 예시
  ~~~ java
  // 11st_method(Class<?> clazz) 와 같은 메서드는 타입 토큰을 인자로 받는 메서드입니다.
  void 11st_method(Class<?> clazz) {
    ...
  }
  
  // 11st_method(String.class)로 호출하면,
  // String.class라는 클래스 리터럴을 타입 토큰 파라미터로 11st_method에 전달합니다.
  11st_method(String.class);
  ~~~

### 3_4. 타입 토큰은 어디에 쓰이나요?
- 주로 타입 토큰은 타입 안전성이 필요한 곳에 사용됩니다.
- 예시
  ~~~ java
  //ObjectMapper의 readValue 메서드 파라미터로 String 과 클래스 리터럴을 전달합니다.
  ProductDto productDto = objectMapper.readValue(jsonString, ProductDto.class);

  //전달된 클래스 리터럴인 ProductDto.class를 타입토큰인 Class<T> valueType로 받고 있습니다.
  public <T> T readValue(String content, Class<T> valueType) {
  ...
  }
  ~~~
### 3_5. 타입 토큰의 한계점은?
- 사례로 THC(Typesafe Heterogenous Container) pattern을 들어 보겠습니다.
- 위 패턴이 적용된 SimpleTypeSafeMap을 만들어 보겠습니다.

~~~ java
public class SimpleTypeSafeMap {
    private Map<Class<?>, Object> map = new HashMap<>();

    public <T> void put(Class<T> k, T v) {
        map.put(k, v);
    }

    public <T> T get(Class<T> clazz) {
        return clazz.cast(map.get(clazz));
    }
}
~~~

- 타입 토큰을 이용해서 별도의 캐스팅 없이도 타입 안전성이 확보가능해졌습니다.

~~~ java
    @Test
    public void type_token을_이용한_put_get_테스트() {
        simpleTypeSafeMap.put(String.class, "11st_문자열");
        simpleTypeSafeMap.put(Integer.class, 11);

        //타입 토큰을 이용해서 별도의 캐스팅 없이도 타입 안전성이 확보가능합니다.
        String v1 = simpleTypeSafeMap.get(String.class);
        Integer v2 = simpleTypeSafeMap.get(Integer.class);

        assertTrue(v1 instanceof String);
        assertTrue(v2 instanceof Integer);
    }
~~~

- 하지만 List<String>.class와 같은 형식의 타입 토큰을 사용할 수 없다는 한계를 가지고 있습니다.
  
~~~ java
    @Test
    public void type_token_한계() {
        simpleTypeSafeMap.put(List.class, Arrays.asList(1,2,3));
        simpleTypeSafeMap.put(List.class, Arrays.asList("1", "2", "3"));

        List<Integer> v1 = (List<Integer>)simpleTypeSafeMap.get(List.class);
        List<String> v2 = (List<String>)simpleTypeSafeMap.get(List.class);

        assertFalse(v1.get(0) instanceof Integer);
        assertTrue(v2.get(0) instanceof String);
    }
~~~
- 이런 한계를 극복할 수 있는 해결책이 바로 수퍼 타입 토큰 입니다. 😁


## 4. 수퍼 타입 토큰(Super Type Token)으로 한계를 극복해 보아요!
### 4_1. 수퍼 타입 토큰이란?
- 타입 토큰계의 슈퍼맨?

![슈퍼맨](https://user-images.githubusercontent.com/18229419/85222296-0de9b180-b3f5-11ea-8bcf-635bc5516c45.png)

- 수퍼 타입 토큰은 Neal Gafter라는 사람이 처음 고안한 방법으로 알려져 있습니다. 수퍼급의 타입 토큰이 아니라, 수퍼 타입을 토큰으로 사용한다는 의미입니다.

<br/>

- 수퍼 타입 토큰은 상속과 Reflection을 기발하게 조합해서 아래 같이, 원래는 사용할 수 없었던 클래스 리터럴을 타입 토큰으로 사용하는 것과 같은 효과를 낼 수 있습니다.

~~~ java
  List<String>.class
~~~
  
- 타입을 구할 수만 있다면 타입 안전성을 확보할 수 있습니다. 하지만, Class<String>와는 달리 Class<List<String>>라는 타입은 위와 같은 클래스 리터럴로 쉽게 구할 수 없습니다.

### 4_2. List 내 타입과 같은 중첩된 타입은 어떻게 구할 수 있나요? 
- Class.getGenericSuperclass()와 ParameterizedType.getActualTypeArguments()를 사용하면 됩니다!
- 쉽게 구할 수 없지만 그래도 방법이 있습니다. [Class.getGenericSuperclass() API 문서](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getGenericSuperclass--)를 보면 아래 정보를 알 수 있습니다.
~~~ java
// Class.class 내 getGenericSuperclass 메서드
  /*
  * 상위의 수퍼 클래스의 타입을 반환하며,
  * 상위의 수퍼 클래스가 ParameterizedType이면, 실제 타입 파라미터들을 반영한 타입을 반환해야 한다.
  * ParameterizedType에 대해서는 별도 문서를 참고하라.  
  */
   public Type getGenericSuperclass() {
        ClassRepository info = this.getGenericInfo();
        if (info == null) {
            return this.getSuperclass();
        } else {
            return this.isInterface() ? null : info.getSuperclass();
        }
    }
~~~

- getGenericSuperclass()를 이용해 출력해보기
~~~java
    @Test
    public void getGenericSuperclass_반환형인_Type_출력() {
        class Super<T> {}
        class MyClass extends Super<List<String>> {}  // 수퍼 클래스에 사용되는 파라미터 타입을 이용한다. 그래서 수퍼 타입 토큰.

        MyClass myClass = new MyClass();

        // 파라미터 타입 정보가 포함된 수퍼 클래스의 타입 정보를 구한다.
        Type typeOfGenericSuperclass = myClass.getClass().getGenericSuperclass();

        // ~~~$1Super<java.util.List<java.lang.String>> 출력됨
        System.out.println(typeOfGenericSuperclass);
    }
~~~

- ParameterizedType.getActualTypeArguments()
- 위에 getGenericSuperclass()의 docs 설명을 보면, 수퍼 클래스가 ParameterizedType이면 타입 파라미터를 포함한 정보를 반환해야 한다고 했으며, ParameterizedType은 별도의 문서를 확인하라고 했습니다.

- [ParameterizedType의 API 문서](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/ParameterizedType.html) 를 보면 Type[] getActualTypeArgumensts()라는 메서드가 있음을 확인할 수 있습니다. 

- getActualTypeArguments()를 이용해 출력해보기
~~~ java
@Test
    public void getActualTypeArguments_Type_출력() {
        class Super<T> {}
        class MyClass extends Super<List<String>> {}  // 수퍼 클래스에 사용되는 파라미터 타입을 이용한다. 그래서 수퍼 타입 토큰.

        MyClass myClass = new MyClass();

        // 파라미터 타입 정보가 포함된 수퍼 클래스의 타입 정보를 구한다.
        Type typeOfGenericSuperclass = myClass.getClass().getGenericSuperclass();

        // ~~~$1Super<java.util.List<java.lang.String>> 출력됨
        System.out.println(typeOfGenericSuperclass);

        // 수퍼 클래스가 ParameterizedType 이므로 ParameterizedType으로 캐스팅 가능
        // ParameterizedType의 getActualTypeArguments()으로 실제 타입 파라미터의 정보를 구함
        Type actualType = ((ParameterizedType) typeOfGenericSuperclass).getActualTypeArguments()[0];

        // 원했던 정보였던 java.util.List<java.lang.String>가 출력됨
        System.out.println(actualType);
    }
~~~

### 4_3. super type token을 이용한 TypeSafeMap 만들어 보아요!
- 이전에 만들었던 SimpleTypeSafeMap은 중첩된 타입토큰을 얻지 못하는 한계가 있었습니다. super type token을 이용해서 한계를 극복한 Map을 만들어 보아요.🤗 
- TypeSafe한 Map을 만들기 위해 Type 정보를 저장할 TypeReference를 만듭니다. TypeSafeMap을 만들어 TypeReference가 가지고 있는 Type을 이용합니다.
- TypeReference

~~~ java
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;

public abstract class TypeReference<T> {
    private final Type type;

    protected TypeReference() {
        Type superClassType = getClass().getGenericSuperclass();
        if (superClassType instanceof ParameterizedType) {
            this.type = ((ParameterizedType)superClassType).getActualTypeArguments()[0];
        } else {
            throw new IllegalArgumentException("TypeReference는 항상 실제 타입 파라미터 정보가 있어야 합니다.");
        }
    }

    public Type getType() {
        return type;
    }
}
~~~

- TypeSafeMap

~~~ java
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.HashMap;
import java.util.Map;

public class TypeSafeMap {
    private final Map<Type, Object> map = new HashMap<>();  // key로 Type을 사용

    public <T> void put(TypeReference<T> k, T v) {
        map.put(k.getType(), v);
    }

    public <T> T get(TypeReference<T> k) {
        final Type type = k.getType();
        final Class<T> clazz;
        if (type instanceof ParameterizedType) {
            clazz = (Class<T>) ((ParameterizedType) type).getRawType();
        } else {
            clazz = (Class<T>) type;
        }
        return clazz.cast(map.get(type));
    }
}
~~~

- 자료형별 put과 get을 테스트하는 코드 작성

~~~ java
import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import main.super_type_token.TypeReference;
import main.super_type_token.TypeSafeMap;

import static org.junit.jupiter.api.Assertions.*;

class TypeSafeMapTest {
    private final TypeSafeMap typeSafeMap = new TypeSafeMap();

    @Test
    public void put_and_get_test_using_string() {
        final String inputData = "문자열";
        final TypeReference<String> tr= new TypeReference<>() {};

        // put
        typeSafeMap.put(tr, inputData);
        // get
        final String outputData = typeSafeMap.get(tr);

        System.out.println("input: " + inputData + "\n" +  "output: " +outputData);
        assertEquals(inputData, outputData);
    }

    @Test
    public void put_and_get_test_using_integer() {
        final Integer inputData = 12345;
        final TypeReference<Integer> tr= new TypeReference<>() {};

        // put
        typeSafeMap.put(tr, inputData);
        // get
        final Integer outputData = typeSafeMap.get(tr);

        System.out.println("input: " + inputData + "\n" +  "output: " +outputData);
        assertEquals(inputData, outputData);
    }

    @Test
    public void put_and_get_test_using_list_string() {
        final List<String> inputData = Arrays.asList("H", "O", "N", "G");
        // List<String>.class와 동일한 효과
        final TypeReference<List<String>> tr= new TypeReference<>() {};

        // put
        typeSafeMap.put(tr, inputData);
        // get
        final List<String> outputData = typeSafeMap.get(tr);

        System.out.println("input: " + inputData + "\n" +  "output: " +outputData);
        assertEquals(inputData, outputData);
    }

    @Test
    public void put_and_get_test_using_list_list_string() {
        final List<List<String>> inputData = Arrays.asList(
            Arrays.asList("H", "O", "N", "G"),
            Arrays.asList("S", "U", "N", "G"),
            Arrays.asList("M", "I", "N")
        );
        // List<List<String>>.class와 동일한 효과
        final TypeReference<List<List<String>>> tr= new TypeReference<>() {};

        // put
        typeSafeMap.put(tr, inputData);
        // get
        final List<List<String>> outputData = typeSafeMap.get(tr);

        System.out.println("input: " + inputData + "\n" +  "output: " +outputData);
        assertEquals(inputData, outputData);
    }

    @Test
    public void put_and_get_test_using_map() {
        final Map<String, String> inputData = new HashMap<>();
        inputData.put("key1", "value1");
        inputData.put("key2", "value2");
        inputData.put("key3", "value3");
        // Map<String, String>.class와 동일한 효과
        final TypeReference<Map<String, String>> tr= new TypeReference<>() {};

        // put
        typeSafeMap.put(tr, inputData);
        // get
        final Map<String, String> outputData = typeSafeMap.get(tr);

        System.out.println("input: " + inputData + "\n" +  "output: " +outputData);
        assertEquals(inputData, outputData);
    }
}
~~~

## 5. 구현이 너무 복잡하네요.. 이미 잘 만들어진건 없을까요? 🥺
### 5_1. Spring의 ParameterizedTypeReference를 사용해주세요! 👏
- TypeReference을 만들기 보다 Spring 횽님의 ParameterizedTypeReference를 사용해 보아요!
- Spring 프레임워크에서도 동일하게 런타임시 발생하는 타입 안정성 문제를 해결하기 위해 ParameterizedTypeReference라는 클래스를 만들었습니다.
- Spring core 패키지에 있는 [ParameterizedTypeReference](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/ParameterizedTypeReference.html)는 아래와 같습니다.

~~~ java
/**
 * The purpose of this class is to enable capturing and passing a generic
 * {@link Type}. In order to capture the generic type and retain it at runtime,
 * you need to create a subclass (ideally as anonymous inline class) as follows:
 *
 * <pre class="code">
 * ParameterizedTypeReference&lt;List&lt;String&gt;&gt; typeRef = new ParameterizedTypeReference&lt;List&lt;String&gt;&gt;() {};
 * </pre>
 *
 * <p>The resulting {@code typeRef} instance can then be used to obtain a {@link Type}
 * instance that carries the captured parameterized type information at runtime.
 * For more information on "super type tokens" see the link to Neal Gafter's blog post.
 *
 * @author Arjen Poutsma
 * @author Rossen Stoyanchev
 * @since 3.2
 * @param <T> the referenced type
 * @see <a href="https://gafter.blogspot.nl/2006/12/super-type-tokens.html">Neal Gafter on Super Type Tokens</a>
 */
public abstract class ParameterizedTypeReference<T> {

	private final Type type;


	protected ParameterizedTypeReference() {
		Class<?> parameterizedTypeReferenceSubclass = findParameterizedTypeReferenceSubclass(getClass());
		Type type = parameterizedTypeReferenceSubclass.getGenericSuperclass();
		Assert.isInstanceOf(ParameterizedType.class, type, "Type must be a parameterized type");
		ParameterizedType parameterizedType = (ParameterizedType) type;
		Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
		Assert.isTrue(actualTypeArguments.length == 1, "Number of type arguments must be 1");
		this.type = actualTypeArguments[0];
	}

	private ParameterizedTypeReference(Type type) {
		this.type = type;
	}


	public Type getType() {
		return this.type;
	}

	@Override
	public boolean equals(@Nullable Object other) {
		return (this == other || (other instanceof ParameterizedTypeReference &&
				this.type.equals(((ParameterizedTypeReference<?>) other).type)));
	}

	@Override
	public int hashCode() {
		return this.type.hashCode();
	}

	@Override
	public String toString() {
		return "ParameterizedTypeReference<" + this.type + ">";
	}


	/**
	 * Build a {@code ParameterizedTypeReference} wrapping the given type.
	 * @param type a generic type (possibly obtained via reflection,
	 * e.g. from {@link java.lang.reflect.Method#getGenericReturnType()})
	 * @return a corresponding reference which may be passed into
	 * {@code ParameterizedTypeReference}-accepting methods
	 * @since 4.3.12
	 */
	public static <T> ParameterizedTypeReference<T> forType(Type type) {
		return new ParameterizedTypeReference<T>(type) {
		};
	}

	private static Class<?> findParameterizedTypeReferenceSubclass(Class<?> child) {
		Class<?> parent = child.getSuperclass();
		if (Object.class == parent) {
			throw new IllegalStateException("Expected ParameterizedTypeReference superclass");
		}
		else if (ParameterizedTypeReference.class == parent) {
			return child;
		}
		else {
			return findParameterizedTypeReferenceSubclass(parent);
		}
	}

}
~~~

## 6. 그렇다면 수퍼 타입 토큰은 주로 어디에 사용할 수 있을까요? 🤔
### 6_1. RestTemplate
- 다른 서버에 있는 자원을 가져오고 싶을 때 Client 라이브러리를 이용하는데요.
- 그 중에서 RestTemplate를 예를 들고 슈퍼 타입 토큰을 이용해 타입 안정성을 확보해보겠습니다.
- 우선 API 2개를 만들고 테스트 코드를 통해 검증해 보겠습니다. 시이작!

~~~ java
import java.util.List;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @RestController
    public static class MyController {
        @GetMapping("/users")
        public List<User> getUsers() {
            return List.of(new User("HONG", 10), new User("SUNG", 11), new User("MIN", 12));
        }

        @GetMapping("/products")
        public List<Product> getProducts() {
            return List.of(new Product("우유", 1000L), new Product("과자", 2000L), new Product("아이스크림", 3000L));
        }
    }

    public static class User {
        private String name;
        private int age;

        private User() { }

        public User(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String getName() {
            return name;
        }

        public int getAge() {
            return age;
        }

        @Override
        public String toString() {
            return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
        }
    }

    public static class Product {
        private String name;
        private long price;

        private Product() {
        }

        public Product(String name, long price) {
            this.name = name;
            this.price = price;
        }

        public String getName() {
            return name;
        }

        public long getPrice() {
            return price;
        }

        @Override
        public String toString() {
            return "Product{" +
                "name='" + name + '\'' +
                ", price=" + price +
                '}';
        }
    }
}
~~~

- 테스트 코드를 작성합니다.
- 테스트 코드에서는 ParameterizedTypeReference를 익명클래스로 계속 만들어 사용중 이지만 실제로 사용하신다면 static하게 한번만 만들고 재사용함 함으로써 성능을 향상시키면 좋을것 같습니다~!👻

~~~ java
import java.util.LinkedHashMap;
import java.util.List;

import org.junit.Assert;
import org.junit.Test;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpMethod;
import org.springframework.web.client.RestTemplate;

import static com.example.demo.DemoApplication.*;

public class ParameterizedTypeReferenceUsageTest {
    private final RestTemplate restTemplate = new RestTemplate();
    private final static String USER_URI = "http://localhost:8080/users";
    private final static String PRODUCT_URI = "http://localhost:8080/products";

    @Test
    public void 슈퍼타입토큰X_user_api_호출해서_사용한_경우() {
        List<User> users = restTemplate.getForObject(USER_URI, List.class);
        final Object first = users.get(0);

        //User에 대한 타입토큰을 알 수 없어서 키벨류 형식을 변환할 수 있는 디폴트 자료형인 LinkedHashMap으로 변환을 했기 때문입니다.
        Assert.assertFalse(first instanceof User);
        Assert.assertTrue(first instanceof LinkedHashMap);
    }

    @Test
    public void 슈퍼타입토큰X__product_api_호출해서_사용한_경우() {
        List<User> users = restTemplate.getForObject(PRODUCT_URI, List.class);
        final Object first = users.get(0);

        //User에 대한 타입토큰을 알 수 없어서 키벨류 형식을 변환할 수 있는 디폴트 자료형인 LinkedHashMap으로 변환을 했기 때문입니다.
        Assert.assertFalse(first instanceof User);
        Assert.assertFalse(first instanceof Product);
        Assert.assertTrue(first instanceof LinkedHashMap);
    }

    @Test
    public void 슈퍼타입토큰O_user_api_호출해서_사용한_경우() {
        List<User> users = restTemplate.exchange(USER_URI,
                                                 HttpMethod.GET,
                                                 null,
                                                 new ParameterizedTypeReference<List<User>>() {})
                                       .getBody();
        final Object first = users.get(0);
        Assert.assertTrue(first instanceof User);

        users.forEach(System.out::println);
    }
}
~~~
### 6_2. ObjectMapper에서도 수퍼 타입 토큰을 사용하고 있어요~

- 스프링에서 제공하는 ParameterizedTypeReference을 사용하지는 않습니다.
- 대신 TypeReference라는 Type을 멤버변수로 가진 추상클래스를 만들어 사용하고 있습니다.

~~~ java
public abstract class TypeReference<T> implements Comparable<TypeReference<T>> {
    protected final Type _type;

    protected TypeReference() {
        Type superClass = this.getClass().getGenericSuperclass();
        if (superClass instanceof Class) {
            throw new IllegalArgumentException("Internal error: TypeReference constructed without actual type information");
        } else {
            this._type = ((ParameterizedType)superClass).getActualTypeArguments()[0];
        }
    }

    public Type getType() {
        return this._type;
    }

    public int compareTo(TypeReference<T> o) {
        return 0;
    }
}
~~~

~~~ java
    /**
     * Method to deserialize JSON content from given JSON content String.
     *
     * @throws JsonParseException if underlying input contains invalid content
     *    of type {@link JsonParser} supports (JSON for default case)
     * @throws JsonMappingException if the input JSON structure does not match structure
     *   expected for result type (or has other mismatch issues)
     */
    public <T> T readValue(String content, TypeReference<T> valueTypeRef)
        throws JsonProcessingException, JsonMappingException
    {
        _assertNotNull("content", content);
        return readValue(content, _typeFactory.constructType(valueTypeRef));
    } 
~~~

### 6_3. feign에서는 사용하지 않아요~!
- 사내에서 Spring Cloud를 사용하고 있고 선언적으로 작성할 수 있는 clinet인 feign을 사용하고 있습니다.
- 혹시나 feign에서도 수퍼 타입 토큰을 사용하는지 알아보게 됐습니다.
- 인터페이스를 통해 정의되는 feign도 역시 데이터를 decode하기 위해 jackson을 사용하고 있습니다.
- jackson에서는 JavaType을 이용해 Tyoe casting을 하고 있습니다.
- JavaType을 만들기 위해서는 역시 Type이 필요합니다. 그래서 Type을 넘겨줄 필요가 있습니다.

~~~ java
Object decode(Response response, Type type) throws IOException {
    try {
      return decoder.decode(response, type);
    } catch (final FeignException e) {
      throw e;
    } catch (final RuntimeException e) {
      throw new DecodeException(response.status(), e.getMessage(), response.request(), e);
    }
  }
~~~

- 하지만 feign은 java.lang.reflect.Method를 이용해 return할 Type을 가져올 수 있기 때문에 슈퍼타입토큰을 사용할 필요가 없습니다. 🤗

~~~ java
 public Type getGenericReturnType() {
        return (Type)(this.getGenericSignature() != null ? this.getGenericInfo().getReturnType() : this.getReturnType());
    }
~~~

---
출처: 
1. [Neal Gafter's blog](http://gafter.blogspot.com/2006/12/super-type-tokens.html)
2. [HomoEfficio. 클래스 리터럴, 타입 토큰, 수퍼 타입 토큰](https://homoefficio.github.io/2016/11/30/%ED%81%B4%EB%9E%98%EC%8A%A4-%EB%A6%AC%ED%84%B0%EB%9F%B4-%ED%83%80%EC%9E%85-%ED%86%A0%ED%81%B0-%EC%88%98%ED%8D%BC-%ED%83%80%EC%9E%85-%ED%86%A0%ED%81%B0/)
3. [토비의 봄 TV - 수퍼 타입 토큰](https://www.youtube.com/watch?v=01sdXvZSjcI)
4. [토비의 봄 TV - 수퍼 타입 토큰(2), 스프링 ResolvableType](https://www.youtube.com/watch?v=y_uGSqpE4So)


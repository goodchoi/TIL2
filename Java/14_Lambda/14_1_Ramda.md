# 14_1_람다

> '자바의 정석 -  남궁성' 14장 학습부분.
> 
> 1.람다식이란?
> 
> 2.람다식 작성하기
> 
> 3.함수형 인터페이스
> 
> 4.java.util.function 패키지
> 
> 5.Function 의 합성과 Predicate 의 결합
> 
> 6.메서드의 참조

## 1. 람다식(Lambda expression)이란?

---

#### 메서드와 함수의 차이

+ 객체 지향에서 함수 대신 객체의 행위나 동작을 의미하는 메서드라는 용어를 사용
  
  + 메서드는 특정 클래스에 속해야한다는 제약이 있음.
  
  + 람다식을 통해 메서드는 제약에서 벗어나서 독립적인 기능을 하기 때문에 '함수' 라는 용어를 사용

+ 함수형 언어로서의 기능
  
  + 기존 객체 지향언어 였던 자바는 자바8의 람다의 등장으로 인해 객체지향언어인 동시에 함수형언어가 됨.

#### 람다식이란

+ 간단히 말해서 메서드를 하나의 식(Expression)으로 표현한 것.

+ 메서드의 매개변수로 전달 되거나 메서드의 결과로 반환 될수 있다. 
  
  + 메서드를 변수처럼 다루는 것이 가능해진것!

## 2. 람다식 작성하기 (기본 문법)

---- 

+ 람다 식은 '익명 함수'이다.
  
  + 메서드에서 이름과 반환 타입을 제거한다.

+ ```java
  int max(int a,int b) {                 (int a, int b) -> {
      return a>b ? a:b;      ->>>>>           return a>b? a:b;
  }                                      }
  ```

+ 매개변수의 타입이 추론 가능한 경우 생략할 수 있고 return문 또한 생략이 가능하다 (이땐 ;를 붙이지 않는다.)   
  
  + 대부분의 경우에 타입은 생략 가능하다. 단, (int a, b) 같이 두 매개변수 중 어느 한 타입 만 생략하는것은 불가능하다.
    
    + ` (int a, int b) -> a > b ? a: b`   --->    ` (a,b) -> a >b ? a : b`
  
  + 매개변수가 하나이면 ()를 생략할 수 있다. 단, 타입이 있으면 생략불가
  
  + 괄호 안의 문장이 하나이면 괄호를 생략할 수 있다. 
  
  + 괄호 안의 문장이 return 문일 경우 괄호를 생략할 수 없다.
    
    + ` (int a) ->{return a * a; }` --->  ` a -> a * a`

## 3. 함수형 인터페이스 (Functional Interface)

----

#### 함수형 인터페이스란?

+ 인터페이스를 통해 람다식을 다루기 위한 인터페이스
  
  + 인터페이스 = 람다식 ;
  
  + 쉽게말해 익명 클래스 구현객체를 생성하는데 이때 람다를 쓰겠다는말

+ 함수형 인터페이스에는<u> 오직 하나의 추상메서드</u>만 연결 되어있어야한다!
  
  + 어노테이션 '@FunctionalInterface'를 붙여 컴파일러가 올바르게 정의 하였는지 확인한다.

+ 예시
  
  + 함수형 인터페이스
  
  ```java
  @FunctionalInterface
  interface MyFunction {
      public abstract int max(int a, int b);
  }
  ```
  
  + 구현
  
  ```java
  //1.람다식을 통한 구현
  MyFunction f1 = (a, b) -> a > b ? a : b;
  f1.max(2, 4);//4
  
  //2.익명 클래스를 통한 구현
  MyFunction f2 = new MyFunction() {
      @Override
      public int max(int a, int b) {
          return a > b ? a : b;
      }
  };
  f2.max(2, 4); //4
  ```

+ 람다식의 타입이 함수형 인터페이스의 타입과 일치 하는것이 아니다.
  
  + 람다식은 익명 객체이고 익명객체에는 타입이 없다.
  
  + 실상은 이러하다.
    
    ` Myfunction f = (MyFunction) (()->{});`
    
    + 람다식은 타입 일치를 위해 이러한 형변환을 허용하고 이때 형변환은 생략되는 것이다.

#### 클래스 변수, 지역 변수 와 람다식

```java
@FunctionalInterface
interface MyFunction {
    void myMethod();
}

class Outer {
    int val = 10;

    class Inner {
        int val = 20;

        void method(int i) {  //(final int i)
            int val = 30; //final int val =30;
            i = 10;    //->에러! 상수의 값을 변경 할 수 없다.
            MyFunction f = () -> {
                System.out.println("i : " + i);
                System.out.println("val : " + val);
                System.out.println("this.val : " + ++this.val);
                System.out.println("Outer.this.val : " + ++Outer.this.val);
            }
        }
    }
}
```

+ 람다식 내 참조하는 지역변수는  모두 상수로 간주 된다. (변경이 허용되지 않는다.)

+ 인스턴스 변수는 상수로 간주 되지 않으므로 값을 변경 할 수 있다.

## 4. java.util.function 패키지

---

#### 자주 쓰이는 함수형 인터페이스

+ java.util.function 에 자주 쓰이는 함수형 인터페이스가 정의 되어있다.
  
  @
  
  + 주요 함수형 인터페이스

+ <img title="" src="file:///Users/choedonghyeon/Library/Application%20Support/marktext/images/2022-12-14-23-59-24-image.png" alt="" width="563">S
  
  (T는 Type을 R은 Return Type을 의미한다. **함수형 인터페이스의 이름들을 보면 의미를 이해하기 쉽다 , 타당한 이유로 명명했기때문이다!)** 

#### Predicate

+ Predicate는 Fuction의 변형이다. 조건식을 람다식으로 표현하는데 사용된다.
- ```java
  Predicate<String> isEmptyStr = s -> s.length() == 0; 
  String s = "";
  
  if(isEmptyStr.test(s)) // if(s.length() == 0)
  {
      ,,,
  }
  ```
  
  - 지금 람다를 배우면서 가장 중요한 것은 객체지향에서 벗어나서 함수를 컨트롤 하고 있다는점이다.

#### 매개변수가 두 개인 함수형 인터페이스

+ 매개 변수를 두개를 받는 의미로 접두어 'Bi' 를 붙인다.

+ | 함수형 인터페이스           | 메서드                    | 설명                                                                        |
  | ------------------- | ---------------------- | ------------------------------------------------------------------------- |
  | BiConsumer<T, U>    | void accept(T t, U u)  | 두 개의 매개변수, 반환 값은 없음. (매개변수를 받지만 반환값이 없기 때문에 소모하는 의미로 consumer라고 명명한것 같다.) |
  | BiPredicate<T, U>   | boolean test(T t, U u) | 두 개의 매개변수, 반환 값은boolean                                                   |
  | BiFunction<T, U, R> | R apply(T t, U u)      | 두 개의 매개변수, 하나의 결과 반환                                                      |

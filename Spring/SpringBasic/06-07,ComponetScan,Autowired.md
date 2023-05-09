> [인프런 김영한님 -스프링 핵심 원리 - 기본편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8)

# 들어가기전에

스프링을 배우기전부터 Autowired 를 아무렇지 않게 써왔다. Autowired가 뭔지 , 어떻게 작동되는지도 알지 못한채 말이다. 이제 이 불편한 느낌을 없애버릴 차례다.

첫 시작은 이렇다. 지금까지 이 강의에서 스프링 빈을 등록할때 @Bean 어노테이션으로 설정 정보에 일일히 나열하여 등록하였다. 근데 등록해야할 빈이 수십,수백개 가된다면 ...?

<u> 개발자는 반복을 싫어한다.</u>

우리는  ```@ComponentScan```, ```@Component``` ,```@Autowired``` 를 배우고 이를 해결할 것이다.

# 6. ComponentScan

## 6-1 기본 원리

+ 설정정보에 `@ComponentScan`을 붙인다. 
  
  ```java
  @Configuration
  @ComponentScan(
          excludeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Configuration.class)
  )
  public class AutoAppConfig {
  
  }
  ```
  
  + 기존과 다르게 @Bean으로 등록한 클래스가 없다.!
  
  + 위의 excludeFilter를 이용해 기존에 `@Configuration`을 붙여놨던 설정정보들을 컴포넌트스캔 대상에서 제외시켰다. 
  
  + 참고로 `@Configuration`도 `@Component`가 내부적으로 포함되어있기때문에 자동으로 스캔의 대상이된다.

+ 다음엔 등록할 빈의 클래스에 `@Component`를 붙인다
  
  ```java
  @Component
  public class MemberServiceImpl implements MemberService {
  
      private final MemberRepository memberRepository;
  
      @Autowired //ac.getBean(MemberRepository.class)
      public MemberServiceImpl(MemberRepository memberRepository) {
          this.memberRepository = memberRepository;
      }
  
      //...
  }
  ```
  
  + `@Component`만 붙인다고 해결되는 것이 아니다.  memberServiceImpl 빈을 등록하기위해서는 필연적으로 생성자를 호출 해야하는데 현재 memberRepository를 의존하고 있다.
  
  + 우리는 의존관계주입을 기존 AppConfig 내부에서 처리했다. 하지만 이젠 `@Autowired`를 붙여 자동으로 처리하게 할것이다.

+ (스프링 부트 시작정보인 `@SpringBootApplication `안에 `@ComponentScan`이 들어있다.)

+ 프로젝트 메인 설정정보는 프로젝트 시작위치에 두는것이 좋다.

+ 애너테이션에는 상속이없다. 애너테이션안에 특정 애너테이션을 들고있는 것을 인식하는것은 스프링의 기능이다.

## 6-2 컴포넌트 대상

+ `@Component` 를 내부적으로 가지는 애너테이션은 모두 대상이된다.

+ ex) `@Controller`, `@Service `, `@Repositroy`  , `@Configuration`

## 6-3 필터

+ IncludeFilters
  
  + 컴포넌트 스캔대상을 추가로 지정한다.

+ excludeFilters
  
  + 컴포넌트 스캔에서 제외할 대상을 지정

+ 사용 예시
  
  + 사용자 애너테이션을 만들어두고
    
    ```java
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface MyExcludeComponent {
    
    }
    ```
  
  + filter를 이렇게 적용한다면
    
    ```java
    @ComponentScan(
                includeFilters = @Filter(type = FilterType.ANNOTATION , classes = MyIncludeComponent.class),
                excludeFilters = @Filter(type = FilterType.ANNOTATION , classes = MyExcludeComponent.class)
        )
        static class ComponentFilterAppConfig {
        }
    ```
  
  + `@MyExcludeComponent`를 가진 클래스의 빈은
    
    ```java
    assertThrows(NoSuchBeanDefinitionException.class,()-> ac.getBean("beanB", BeanB.class));
    ```
  
  + 이런 예외를 던질것이다.

+ `@Component` 면 충분하기 때문에, excludeFilters는 사용빈도가 매우낮다. 또한, 이런 옵션을 변경하기보다는 스프링의 기본설정에 최대한 맞추어 사용한는것을 권장한다.

## 6-4 참고

+ 수동 빈등록 과 자동 빈등록의 우선순위는 수동 빈등록이 높다.

---

# 7 의존관계 자동주입

## 7-1 의존관계 주입 방법의 종류

+ 생성자주입

+ 수정자 주입(setter주입)

+ 필드 주입

+ 일반 메서드 주입

## 7-2 생성자 주입

+ 특징 
  
  + 생성자 호출 시점에 딱 1번만 호출되는 것이 보장
  
  + <mark>불변, 필수 </mark>의존관계에 사용

+ 예시
  
  ```java
      @Autowired
      public OrderServiceImpl(MemberRepository memberRepository,@MainDiscountPolicy DiscountPolicy discountPolicy) {
          this.memberRepository = memberRepository;
          this.discountPolicy = discountPolicy;
      }
  ```

+ 생성자 **1개**일시 `@Autowired`를 생략할수있다. (스프링빈에만 해당)

## 7-3 수정자 주입(setter 주입)

+ 특징
  
  + <mark>선택, 변경</mark> 가능성이 있는 의존관계에 사용
    
    + 필드에 final을 붙일수없다!
  
  + 자바빈 프로퍼티 규약의 수정자 메서드 방식을 사용

+ 예시
  
  ```java
  @Component
  
  public class OrderServiceImpl implements OrderService {
  
      private  MemberRepository memberRepository;
      private  DiscountPolicy discountPolicy; 
  
      @Autowired
      public void setMemberRepository(MemberRepository memberRepository) {
          this.memberRepository = memberRepository;
      }
  
      @Autowired
      public void setDiscountPolicy(DiscountPolicy discountPolicy) {
          this.discountPolicy = discountPolicy;
      }
  ```

## 7-4 필드 주입

+ 특징
  
  + 코드가 간결하나 테스트하기 힘들다.
  
  + DI프레임워크 없이 아무것도 할수없다.
  
  + **Deprecated**
  
  + `@Configuration`과 같은 특별한 용도로만 사용
  
  + *그냥 사용하지말자!*

+ 예제
  
  ```java
  public class OrderServiceImpl implements OrderService {
  
      @Autowired
      private  MemberRepository memberRepository;
      @Autowired
      private  DiscountPolicy discountPolicy; 
  ```

## 7-5 일반 메서드 주입

+ 한번에 여러 필드 주입 가능

+ 일반적으로 잘사용하지 않음

## 7-6 옵션 처리

스프링 빈이 없어도 동작해야할 때가 있다. 즉 있으면 쓰고 없으면 그냥 진행

+ `@Autowired`의 required의 기본값이 true이기 때문에 빈이없으면 오류난다.

+ 옵션처리법
  
  + `@Autowired(required=false)`
    
    + 주입할 대상이 없으면 메서드 호출이 안됨 (생성자에는 사용불가)
  
  + `@Nullable`
    
    + 주입할 대상이없으면 null입력
  
  + `Optional<>`
    
    + 주입할 대상 없으면 Optional.empty입력
  
  + 밑의 두 방법은 생성자 주입에서도 사용이가능하다.

## 7-7 왜 생성자 주입법을 주로 사용하는가?

+ **불변**
  
  + 의존관계주입이 일어나면 프로그램종료시까지 거의 변경할 일이없다. 오히려 변하면 안된다.
  
  + 수정자 주입은 setXXX메소드를 필연적으로 public으로 열어나야 하는데 이는 바람직하지않다.
  
  + 딱 1번만 호출되므로 불변하게 설계가 가능하다.

+ **누락**
  
  + 프레임워크 없이 순수 자바코드로 테스트할때 어떤 의존관계를 주입해야하는지 쉽게 알지 못한다.(IDE를 통해서)

+ final 키워드 사용가능
  
  + 필드에 final 키워드를 붙일 수 있고 , 컴파일 오류를 잡을 수있다.!

> > *컴파일 오류는 세상에서 가장빠르고 좋은오류이다...*

## 7-9 롬복 (lombok)

요즘 필수 라이브러리 ,, 롬복에 대해 알아보자 (getter,sette 줄이는 용으로만 썼던나를 반성하며)

+ 생성자 1개일 시 Autowired 를 생략할 수 있다고 했다. 그럼 이런 코드도 가능하다.
  
  ```java
  @Component
  @RequiredArgsConstructor //final이 붙은 필드를 매개변수로 받는 생성자를 만들어준다. 깔끔하고 쉽고 좋다 그냥 좋다
  public class OrderServiceImpl implements OrderService {
  
      private final MemberRepository memberRepository;
      private final DiscountPolicy discountPolicy; //DIP 위반 방지를 위하여 인터페이스만 의존하도록 설계
  
  }
  ```
  
  + `@Autowired`날리고 생성자 까지 날렸다. 롬복의 힘...

### 롬복 추가하기

+ `build.gradle`에 코드 추가
  
  ```java
  //lombok 설정 추가 시작
  configurations {
      compileOnly{
          extendsFrom annotationProcessor
      }
  }
  //lombok 설정 추가 끝
  
  dependencies {
  implementation 'org.springframework.boot:spring-boot-starter'
  //lombok추가 시
  compileOnly 'org.projectlombok:lombok'
  annotationProcessor 'org.projectlombok:lombok'
  testCompileOnly 'org.projectlombok:lombok'
  testAnnotationProcessor 'org.projectlombok:lombok'
  //lombok추가 끝
  testImplementation 'org.springframework.boot:spring-boot-starter-test
  }
  ```
  
  + 코끼리 누를것.

## 7-10 조회 빈이 두개 이상이라면?

  `@Autowired`는 기본적으로 타입으로 조회한다. 그렇다면 똑같은 타입의 빈이 두개 이상이면?

```java
 NoUniqueBeanDefinitionException: No qualifying bean of type
 'hello.core.discount.DiscountPolicy' available: expected single matching bean
 but found 2: fixDiscountPolicy,rateDiscountPolicy
```

+ 이런 오류를 확인할 수 있다.

+ 그렇다고 하위타입으로 지정하는것은 DIP를 위반하는일이다.

+ 수동 주입을 하면 이럴 걱정이 없지만 자동주입으로도 해결할 수있다.

+ 해결방법
  
  + `@Autowired` 필드 명 매칭  
    
    + 필드이름을 빈이름이랑 똑같이 맞춰주면 된다. (ㅋㅋ) 뭔가 좀 촌스런 방법인데? 
  
  + `@Qualifier`  `@Qualifier`끼리 매칭 빈 이름 매칭
    
    ```java
      @Component
      @Qualifier("mainDiscountPolicy")
      public class RateDiscountPolicy implements DiscountPolicy {}
    ```
    
    ```java
    @Autowired
      public OrderServiceImpl(MemberRepository memberRepository,
                              @Qualifier("mainDiscountPolicy") DiscountPolicy
      discountPolicy) {
          this.memberRepository = memberRepository;
          this.discountPolicy = discountPolicy;
    }
    ```
    
    + 딱히 설명은 생략한다. 참고로 잘못적었을시 해당 문자열의 빈을 찾는 어이없는 과정을 하게
      끔 만들어놨다..
    
    + 보다시피 Quailfier를 한번쓰면 모든 코드에 Qualifier를 붙여야하는 상황을 만들것이다.
  
  + `@Primary` 사용
    
    ```java
      @Component
      @Primary
      public class RateDiscountPolicy implements DiscountPolicy {}
    ```
    
    + 이런 식으로 사용하여 우선순위를 부여한다.
    
    + 제일 심플하고 간단한것 같다.

+ 최적의 사용법
  
  + ex) 메인데이터베이스 커넥션과 특별한 데이터베이스커넥션이 있다고 했을때 메인 데이터베이스  빈은`@Primary`를 붙이고, 특별한 데이터베이스 커넥션에는`@Qualifier`를 이용하면 깔끔하게 유지할 수 있다. 정확히 이해가 가는 부분이다.
  
  + 물론 이때도 Qualifier가 우선순위가 더높다. (스프링에선 수동이 무조건 우선순위가 높다.)

## 7-11 애너테이션 직접만들기

`@Qualifier`에 문자열을 직접 표기하는것은 너무 쿨하지 못하다. 이럴땐 이것도 괜찮다.

```java
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@Qualifier("mainDiscountPolicy")
public @interface MainDiscountPolicy {
}
```

+ 이런식으로 @Qualifer를 임의 포함하는 임의의 애너테이션을 직접 만들었다.

+ 그리고 의존관계 주입하는 곳에서

```java
@Autowired
public OrderServiceImpl(MemberRepository memberRepository,
                          @MainDiscountPolicy DiscountPolicy discountPolicy) {
      this.memberRepository = memberRepository;
      this.discountPolicy = discountPolicy;
  }
```

+ 이런식으로 애너테이션을 붙여서 사용한다.

+ 이제좀 쿨한것같다.

## 7-12 특정 클래스의 모든 빈이 필요할 때

+ ex) 할인서비스를 제공하는데 클라이언트가 할인의 종류를 선택할 수 있는경우 (rate,fix)

+ 스프링을 이용하여 매우 쉽게 구현가능 하다.

+ List와 Map을 사용하면 가능하다.!
  
  ```java
   @Autowired
          public DiscountService(Map<String, DiscountPolicy> policyMap, List<DiscountPolicy> policies) {
              this.policyMap = policyMap;
              this.polcies = policies;
              System.out.println("policyMap = " + policyMap);
              System.out.println("polcies = " + polcies);
          }
  ```

    이렇게 하면 Map에 키값으로 빈의 이름 밸류에는 빈객체가 담기게 된다. LIst에는빈객체만 담긴다.

## 7-13 자동, 수동의 올바른 실무 운영기준

+ 편리한 자동 기능을 기본 으로 사용하자.
  
  + 최근 스프링부트는 컴포넌트 스캔을 기본으로 사용하고, 자동으로 계속 옮겨나는 추세다.
  
  + 자동 빈등록을 해도 OCP,DIP 를 지킬 수 있는데 굳이 특정 상황이 아닌데 수동을 쓸 필요는 없다.

+ 그럼 수동은 언제?
  
  + **업무로직 빈**은 숫자가 매우많고 유사한 패턴이 많기 때문에 자동을 적극적으로 이용해야한다.
  
  + 주로 기술 적인문제나 공통관심사 처리할떄 수동 빈등록을 사용하는것이 좋다. 
    
    + ex) 데이터 베이스 연결 , 공통 로그처리

## 7-14 이번 장에서 배운점

스프링 부트를 이용해서 프로젝트를 할때 스프링에 대한 나의 이해도가 얼마나 형편 없었는지 알게되었다. Autowired를 왜써야 했고, 어떤 원리로 동작하는지 전혀 몰랐다. 사실 원초적인 문제는 스프링이 대체 뭘하는 건지 몰랐다는 사실이지만... 이제라도 하나씩 알아가는것 같아서 너무 다행이다. 

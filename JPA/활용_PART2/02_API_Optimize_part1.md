[인프런 - 김영한님 강의 스프링 부트와 JPA 활용 part2](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94)

# 2. API 개발고급 - 지연로딩 조회 성능 최적화

> 매우 매우 매우 매우 중요 하다 200% 이해할것.

## 2-1 version1 엔티티 직접 노출

```java
@RestController
@RequiredArgsConstructor
public class OrderSimpleApiController {

    private final OrderRepository orderRepository;

    @GetMapping("/api/v1/simple-orders")
    public List<Order> orderV1() {
        List<Order> all = orderRepository.findAll(new OrderSearch());
        for (Order order : all) {
            order.getMember().getName();//Lazy 강제 초기화
            order.getDelivery().getAddress();
        }
        return all;
    }
}
```

+ 아무 작업 안하고 이 메서드가 호출되면 오류가 난다 왜?
  
  + 지연로딩을 걸어놓은 상태이므로 member 와 delivery는 프록시 객체이다. `jackson` 라이브러리는 프록시를 이해하지 못한다
  
  + -> 예외 발생

+ `Hibernate5Modeu`을 빈으로 등록하면 해결 되긴 한다.

#### 하이버네이트 모듈 등록

+ `build.gradle` 에 라이브러리 추가
  
  + `implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5'`

+ `JpashopApplication` 에 코드 추가

```java
@SpringBootApplication
public class JpashopApplication {

    public static void main(String[] args) {
        SpringApplication.run(JpashopApplication.class, args);
    }

    @Bean
    Hibernate5Module hibernate5Module() {
        Hibernate5Module hibernate5Module = new Hibernate5Module();
        hibernate5Module.configure(Hibernate5Module.Feature.FORCE_LAZY_LOADING,true);
        return hibernate5Module;
    }
}
```

+ `Force_LAZY_LOADING` 옵션을 키면 양방향 연관관계를 계속로딩 한다. (무한루프) 따라서 @JsonIgnore를 양방향 중 한곳에 줘야한다.

+ 정리
  
  + 버전1에선 엔티티 자체를 외부에 노출 시키고있다는 점에서 이미 탈락이다.
  
  + 또한 @JsonIgnore 같이 어이없는 어노테이션을 엔티티에 달아야하기때문에 제약사항이 너무 많다.

<br>

## 2-2 version2 엔티티를 DTO로 반환하기

```java
    @GetMapping("/api/v2/simple-orders")
    public List<SimpleOrderDto> ordersV2() {
        List<Order> orders = orderRepository.findAll(new OrderSearch());
        List<SimpleOrderDto> result = orders.stream()
                .map(SimpleOrderDto::new)
                .collect(toList());

        return result;
    }
    
    @Data
    static class SimpleOrderDto {
        private Long orderId;
        private String name;
        private LocalDateTime orderDate;
        private OrderStatus orderStatus;
        private Address address;

        public SimpleOrderDto(Order order) { //dto가 엔티티를 파라미터로 받는것은 크게 문제가 없음
            orderId = order.getId();
            name = order.getMember().getName();
            orderDate = order.getOrderDate();
            orderStatus = order.getStatus();
            address = order.getDelivery().getAddress();
        }
    }
```

+ v1 과 달리 엔티티를 DTO로 변환했다. DTO를 사용해서 얻을 수 있는 이점은 앞장에서 이미 설명했다.

+ 이렇게 한다고해서 쿼리수가 적게 나가지 않는다 . 여전히 (N+1) 문제가 발생한다. 지연로딩 걸려있는 필드수가 많을수록 N은 반복해서 늘어난다.

<br>

## 2-3 version3 페치 조인으로 최적화 하기

```java
    @GetMapping("/api/v3/simple-orders")
    public List<SimpleOrderDto> ordersV3() {
        List<Order> orders = orderRepository.findAllwithMemberDelivery();
        List<SimpleOrderDto> result = orders.stream()
                .map(SimpleOrderDto::new)
                .collect(toList());
        return result;
    }
```

+ 엔티티를 DTO로 변환하고 페치 조인을 이용했다.



`orderRepository.findAllwithMemberDelivery()` 

```java
 public List<Order> findAllwithMemberDelivery() {
        List<Order> resultList = em.createQuery("select o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d", Order.class
                ).getResultList();

        return resultList;
    }
```

이렇게 페치 조인을 사용했을때 실제 나가는 쿼리문은 다음과 같다.

```sql
select
        order0_.order_id as order_id1_6_0_,
        member1_.member_id as member_i1_4_1_,
        delivery2_.delivery_id as delivery1_2_2_,
        order0_.delivery_id as delivery4_6_0_,
        order0_.member_id as member_i5_6_0_,
        order0_.order_date as order_da2_6_0_,
        order0_.status as status3_6_0_,
        member1_.city as city2_4_1_,
        member1_.street as street3_4_1_,
        member1_.zipcode as zipcode4_4_1_,
        member1_.name as name5_4_1_,
        delivery2_.city as city2_2_2_,
        delivery2_.street as street3_2_2_,
        delivery2_.zipcode as zipcode4_2_2_,
        delivery2_.delivery_status as delivery5_2_2_ 
    from
        orders order0_ 
    inner join
        member member1_ 
            on order0_.member_id=member1_.member_id 
    inner join
        delivery delivery2_ 
            on order0_.delivery_id=delivery2_.delivery_id
```

+ N+1 문제가 해결되었다. 단한개의 쿼리문이 나간다.

<br> 

## 2-4 version4 JPA에서 DTO 바로 조회하기

```java
    @GetMapping("/api/v4/simple-orders")
    public List<SimpleOrderQueryDto> ordersV4() {
        List<SimpleOrderQueryDto> result = orderRepository.findOrderDtos();
        return result;
    }
```

+ 위와 다르게 `orderRepository` 의 메서드를 호출하는데 이때 API 스펙에 맞는 Dto 를 바로 반환해주는 메서드를 정의한다.



`orderRepository.findOrderDtos()` 

```java
public List<SimpleOrderQueryDto> findOrderDtos() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.SimpleOrderQueryDto(o.id,m.name,o.orderDate,o.status,d.address) from Order o" +
                " join o.member m" +
                " join o.delivery d", SimpleOrderQueryDto.class
        ).getResultList();
    }
```

+ `SimpleOrderQueryDto`를 새로 정의했다. 이미 똑같은 아이가 컨트롤러의 내부클래스에 정의 되어있는데 굳이 새로 정의한이유는 리포지토리가 컨트롤러에 의존하는 상황이 발생하기 때문이다.

> 컨트롤러 -> 서비스 -> 리포지토리  처럼 의존관계는 단방향으로 흘러가야한다.

+ 이때, **dto 는 엔티티가 아니므로, 페치 조인을 사용할 수 없고**

+ `new` 명령어를 사용해서 JPQL 결과를 DTO로 즉시 변환한다.

+ 전체 패키지명 모두 명시해야하며, 모든 인자를 매개변수로 받는 생성자가 정의 되어있어야한다.

+ v3 와 v4의 경우 최종 쿼리문은 select절만 다르고 Join은 같다.

<br>

## 2-5 결론 :  V3 vs V4 무엇을 선택해야 하는가?

#### v3 - fetch join

+ v3 의 장점
  
  + 재사용성이 크다.
  
  + 쿼리가 간단하다.

+ 단점 
  
  + 필요없는 필드까지 모두 조회한다. 하지만 이로 인한 영향은 그렇게 크지않다

### v4 - 일반 조인 + dto 바로 반환

+ 장점
  
  + 원하는 값을 선택해서 조회가 가능하다.

+ 단점
  
  + 재사용성이 떨어진다. 



> **쿼리 방식 선택 권장 순서**
> 
> 1. 엔티티를 DTO로 변환하는 방법 선택
> 
> 2. 필요시 페치 조인으로 성능 최적화 -> 여기서 대부분의 성능 이슈 해결
> 
> 3. 그래도 성능이 잘 안나온다. DTO 직접 조회
> 
> 4. 여기까지 했는데도 빡세다 -> 네이티브 SQL 혹은 스프링 JDBC Template를 사용해서 직접 SQL을 사용한다. ----------------- 최후의 보루





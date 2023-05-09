[인프런 - 김영한님 강의 스프링 부트와 JPA 활용 part2](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94)

# 3 API 개발 고급 - 컬렉션 조회 최적화 (ver1 ~ ver3) : 엔티티 조회

## 3-1 [version1] 엔티티 직접 노출

```java
@RestController
@RequiredArgsConstructor
public class OrderApiController {

    private final OrderRepository orderRepository;

    @GetMapping("/api/v1/orders")
    public List<Order> ordersV1() {
        List<Order> all = orderRepository.findAll(new OrderSearch());
        for (Order order : all) {
            order.getMember();
            order.getDelivery().getAddress();
            List<OrderItem> orderItems = order.getOrderItems();
            orderItems.forEach(o -> o.getItem().getName());
        }
        return all;
    }
}
```

+ 직접 프록시 초기화를 하나씩 해주는 모습

+ `Hibernate5Module` 에 의해 지연로딩된 엔티티도 JSON으로 출력하게 한다.

+ 이때, 양방향 한곳에 `@JsonIgnore` 를 추가해야한다.

+ 엔티티를 직접 노출하므로 좋은 방법이 아니다.

<br>

## 3-2 [version2]엔티티를 DTO로 변환

```java
@GetMapping("/api/v2/orders")
public List<OrderDto> ordersV2() {
     List<Order> orders = orderRepository.findAll();
     List<OrderDto> result = orders.stream()
         .map(o -> new OrderDto(o))
         .collect(toList());
     return result;
}
```

+ 참고 ) List 를 사용할때는 `stream`을 적극 사용하자! 이제 할줄안다 나도,..

#### DTO

```java
    @Data
    static class OrderDto {
        private Long orderId;
        private String name;
        private LocalDateTime orderDate; //주문시간
        private OrderStatus orderStatus;
        private Address address;
        private List<OrderItemDto> orderItems;

        public OrderDto(Order order) {
            orderId = order.getId();
            name = order.getMember().getName();
            orderDate = order.getOrderDate();
            orderStatus = order.getStatus();
            address = order.getDelivery().getAddress();
            orderItems = order.getOrderItems().stream()
                    .map(orderItem -> new OrderItemDto(orderItem))
                    .collect(toList());
        }
    }
    @Data
    static class OrderItemDto {
        private String itemName;//상품 명
        private int orderPrice; //주문 가격
        private int count; //주문 수량
        public OrderItemDto(OrderItem orderItem) {
            itemName = orderItem.getItem().getName();
            orderPrice = orderItem.getOrderPrice();
            count = orderItem.getCount();
        }
    }
```

+ `OrderDto`  필드 중에 `List<OrderItemDto>` 가 있다. Dto로 감쌌음에도 필드에 또 엔티티가 있다. 이때 Json으로 만들면 , 또 엔티티가 노출되게 된다. 
  
  + 즉, Dto를 감쌀때, 필드에 엔티티가 있으면 안된다. 이것 또한 다시 Dto로 감싸 주어야한다.

+ **version1 과 version2는 정확히 지연로딩을 사용하고 있다.**
  
  + version1에서는 직접 프록시 초기화를 사용했고,
  
  + version2에서는 Dto생성자 안에서  `getter` 메소드를 통해 프록시 초기화가 일어난다.

+ 지연 로딩은 언제나 그렇듯 `N+1`  문제를 발생시킨다. 지연로딩되는 엔티티가 많을수록 N은 점점 늘어 날것이다.

#### 반환된 JSON 결과

```json
[
    {
        "orderId": 4,
        "name": "userA",
        "orderDate": "2023-04-24T12:43:43.090356",
        "orderStatus": "ORDER",
        "address": {
            "city": "서울",
            "street": "1",
            "zipcode": "1111"
        },
        "orderItems": [
            {
                "itemName": "JPA BOOK1",
                "orderPrice": 10000,
                "count": 1
            },
            {
                "itemName": "JPA BOOK2",
                "orderPrice": 20000,
                "count": 2
            }
        ]
    },
    {
        "orderId": 11,
        "name": "userB",
        "orderDate": "2023-04-24T12:43:43.114651",
        "orderStatus": "ORDER",
        "address": {
            "city": "진주",
            "street": "2",
            "zipcode": "2222"
        },
        "orderItems": [
            {
                "itemName": "SPRING BOOK1",
                "orderPrice": 20000,
                "count": 3
            },
            {
                "itemName": "SPRING BOOK1",
                "orderPrice": 40000,
                "count": 4
            }
        ]
    }
]
```

<br>

## 3-3 [version3] 엔티티를 DTO로 변환 -> 페치 최적화

```java
@GetMapping("/api/v3/orders")
public List<OrderDto> ordersV3() {
     List<Order> orders = orderRepository.findAllWithItem();
     List<OrderDto> result = orders.stream()
        .map(o -> new OrderDto(o))
        .collect(toList());
     return result;
}
```

+ version2와 리포지토리 메서드 빼곤 정확히 일치한다. -> 즉 메서드 내부에서 사용하는 쿼리문만 수정하겠다는 뜻

#### orderRepository.findAllWithItem()

```java
public List<Order> findAllWithItem() {
     return em.createQuery(
         "select distinct o from Order o" +
         " join fetch o.member m" +
         " join fetch o.delivery d" +
         " join fetch o.orderItems oi" +
         " join fetch oi.item i", Order.class)
     .getResultList();
}
```

+ 페치조인으로 한번의 쿼리문만 나간다.

+ distinct를 사용해서 데이터베이스 row수를 줄여준다. -> 현재상황에서는 어플리케이션 레벨에서 적용된다.

#### 실제 쿼리문

```sql
     select
        distinct order0_.order_id as order_id1_6_0_,
        member1_.member_id as member_i1_4_1_,
        delivery2_.delivery_id as delivery1_2_2_,
        orderitems3_.order_item_id as order_it1_5_3_,
        item4_.item_id as item_id2_3_4_,
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
        delivery2_.delivery_status as delivery5_2_2_,
        orderitems3_.count as count2_5_3_,
        orderitems3_.item_id as item_id4_5_3_,
        orderitems3_.order_id as order_id5_5_3_,
        orderitems3_.order_price as order_pr3_5_3_,
        orderitems3_.order_id as order_id5_5_0__,
        orderitems3_.order_item_id as order_it1_5_0__,
        item4_.name as name3_3_4_,
        item4_.price as price4_3_4_,
        item4_.stock_quantity as stock_qu5_3_4_,
        item4_.artist as artist6_3_4_,
        item4_.etc as etc7_3_4_,
        item4_.author as author8_3_4_,
        item4_.isbn as isbn9_3_4_,
        item4_.actor as actor10_3_4_,
        item4_.director as directo11_3_4_,
        item4_.dtype as dtype1_3_4_ 
    from
        orders order0_ 
    inner join
        member member1_ 
            on order0_.member_id=member1_.member_id 
    inner join
        delivery delivery2_ 
            on order0_.delivery_id=delivery2_.delivery_id 
    inner join
        order_item orderitems3_ 
            on order0_.order_id=orderitems3_.order_id 
    inner join
        item item4_ 
            on orderitems3_.item_id=item4_.item_id
```

+ 페치 조인을 사용했으므로 당연히 페이징 처리가 불가능하다.

+ 또, 컬렉션 페치 조인을 1개만 사용할 수있다. 

<br>

## 3-4 [version 3.1] 엔티티를 DTO로 변환 - 페이징 한계 돌파

기본편에서 이미 배웠지만,,,컬렉션 엔티티 조회와 페이징 처리를 한번에 하려면 어떻게 해야할 까?

#### 한계 돌파

+ 가장 먼저 `XXXToOne` 관계를 모두 페치 조인한다. -> 페이징 쿼리에 영향을 주지 않으므로

+ 컬렉션은 지연로딩으로 조회한다.

+ 이때, 성능의 최적화를 위해 `hibernate.default_batch_fetch_size` 혹은 `@BatchSize` 를 적용한다.
  
  + `hibernate.default_batch_fetch_size` : 글로벌 설정
  
  + `@BatchSize` : 개별 설정

```java
@GetMapping("/api/v3.1/orders")
public List<OrderDto> ordersV3_page(@RequestParam(value = "offset",
    defaultValue = "0") int offset,
 @RequestParam(value = "limit", defaultValue= "100") int limit) {

     List<Order> orders = orderRepository.findAllWithMemberDelivery(offset,limit);
     List<OrderDto> result = orders.stream()
         .map(o -> new OrderDto(o))
         .collect(toList());
     return result;
}
```

+ 페이징 처리를 위해 `offset` 과 `limit`을 파라미터로 받는다.

#### orderRepository.findAllWithMemberDelivery(offset,limit)

```java
public List<Order> findAllWithMemberDelivery(int offset, int limit) {
     return em.createQuery(
             "select o from Order o" +
             " join fetch o.member m" +
             " join fetch o.delivery d", Order.class)
         .setFirstResult(offset)
         .setMaxResults(limit)
         .getResultList();
}
```

+ 정확히 `xxxToOne`  관계만 페치 조인을 사용했다. 그리고 페이징 처리를 했다. 그리고 batch size를 설정 했다. 끝이다.

#### appication.yaml

```yaml
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/jpashop
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
#        show_sql: true
        format_sql: true
        default_batch_fetch_size: 100 /////////////////////////////////

logging:
  level:
    org.hibernate.SQL: debug
    #  org.hibernate.type: trace
```

#### 실제 쿼리문

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
            on order0_.delivery_id=delivery2_.delivery_id limit ? offset ?
    ----   ----   ----   ----   ----   ----   ----   ----   ----   ----
        select
        orderitems0_.order_id as order_id5_5_1_,
        orderitems0_.order_item_id as order_it1_5_1_,
        orderitems0_.order_item_id as order_it1_5_0_,
        orderitems0_.count as count2_5_0_,
        orderitems0_.item_id as item_id4_5_0_,
        orderitems0_.order_id as order_id5_5_0_,
        orderitems0_.order_price as order_pr3_5_0_ 
    from
        order_item orderitems0_ 
    where
        orderitems0_.order_id=?  

    --------------------------------------------------------------------
    select
        item0_.item_id as item_id2_3_0_,
        item0_.name as name3_3_0_,
        item0_.price as price4_3_0_,
        item0_.stock_quantity as stock_qu5_3_0_,
        item0_.artist as artist6_3_0_,
        item0_.etc as etc7_3_0_,
        item0_.author as author8_3_0_,
        item0_.isbn as isbn9_3_0_,
        item0_.actor as actor10_3_0_,
        item0_.director as directo11_3_0_,
        item0_.dtype as dtype1_3_0_ 
    from
        item item0_ 
    where
        item0_.item_id in (
            ?, ?
        )
```

+ 쿼리문은 세번 나갔다. 예상했던 결과다. 

+ 이 방식의 장점은 페이징 처리가 가능하며, DB 데이터 전송량이 최적화된다는 것이다.(중복 Row수가 없어짐)

> `default_batch_fetch_size 의` 크기는 적당한 사이즈를 골라야 하는데 보통 100~1000사이를 선택한다. 1000으로 했을때 순간 부하가 증가할 수 있다. 100이나 1000이나 결국 데이터를 로딩하므로 메모리 사용량이 같다. 순간 부하를 어디까지 견딜수 있는지로 결정하면 된다.

> 다음장에서는 JPQL문 안에서 엔티티로 조회 하는것이아닌 DTO로 바로 조회하는 방법을 다룬다.

[인프런 - 김영한님 강의 스프링 부트와 JPA 활용 part2](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94)

# 4 API 개발 고급 - 컬렉션 조회 최적화 (ver4 ~ ver6) : DTO 조회

## 4-1 [version4] JPA에서 DTO 직접 조회

```java
private final OrderQueryRepository orderQueryRepository;

@GetMapping("/api/v4/orders")
public List<OrderQueryDto> ordersV4() {
 return orderQueryRepository.findOrderQueryDtos();
}
```

+ 특정 쿼리를 사용하는 패키지를 따로 생성했다. 

#### OrderQueryRepository

```java
//특정화면에 핏한, 혹은 API 완전 맞춤형 쿼리들은 따로 패키지로 만드는것이 좋다.
@Repository
@RequiredArgsConstructor
public class OrderQueryRepository {

    private final EntityManager em;


    public List<OrderQueryDto> findOrderQueryDtos() {
        List<OrderQueryDto> result = findOrders();

        result.forEach(o -> {
            List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId());
            o.setOrderItems(orderItems);
        });
        return result;
    }

    private List<OrderQueryDto> findOrders() {
        return em.createQuery(
                        "select new jpabook.jpashop.repository.order.query.OrderQueryDto(o.id,m.name,o.orderDate,o.status,d.address)" +
                                " from Order o" +
                                " join o.member m" +
                                " join o.delivery d", OrderQueryDto.class)
                .getResultList();
    }

    private List<OrderItemQueryDto> findOrderItems(Long orderId) {

        return em.createQuery(
                        "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id,i.name,oi.orderPrice,oi.count)" +
                                " from OrderItem oi" +
                                " join oi.item i" +
                                " where oi.order.id = :orderId", OrderItemQueryDto.class)
                .setParameter("orderId", orderId)
                .getResultList();

    }
}
```

#### OrderQueryDto

```java
@Data
@EqualsAndHashCode(of = "orderId")
public class OrderQueryDto {


    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;
    private List<OrderItemQueryDto> orderItems;

    public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
    }

    public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, List<OrderItemQueryDto> orderItems) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
        this.orderItems = orderItems;
    }
}
```

#### OrderItemQueryDto

```java
@Data
@EqualsAndHashCode
public class OrderItemQueryDto {

    private Long orderId;
    private String itemName;
    private int orderPrice;
    private int count;

    public OrderItemQueryDto(Long orderId, String itemName, int orderPrice, int count) {
        this.orderId = orderId;
        this.itemName = itemName;
        this.orderPrice = orderPrice;
        this.count = count;
    }
}
```

+ 먼저 `xxxToOne`관계를 먼저 조회 하고, id 값으로 `OrderItemQueryDto` 를 조회 해서 set해준다. 

+ 즉 메서드 흐름은 조회 쿼리 메서드 2개 + 합치는 메서드(컨트롤러 역할) 한개이다.

+ 쿼리는 1 + N 번 나가게 된다. 

<br>

## 4-2 [version5] JPA에서 DTO 직접 조회 - 컬렉션 조회 최적화 -추천요~

```java
@GetMapping("/api/v5/orders")
public List<OrderQueryDto> ordersV5() {
     return orderQueryRepository.findAllByDto_optimization();
}
```



#### OrderQueryRepository

```java
public List<OrderQueryDto> findAllByDto_optimization() {

        List<OrderQueryDto> result = findOrders();

        Map<Long, List<OrderItemQueryDto>> orderItemMap = findOrderItemMap(toOrderIds(result));

        result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

        return result;
    }

    private Map<Long, List<OrderItemQueryDto>> findOrderItemMap(List<Long> orderIds) {
        List<OrderItemQueryDto> orderItems = em.createQuery(
                        "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id,i.name,oi.orderPrice,oi.count)" +
                                " from OrderItem oi" +
                                " join oi.item i" +
                                " where oi.order.id in :orderIds", OrderItemQueryDto.class)
                .setParameter("orderIds", orderIds)
                .getResultList();

        Map<Long, List<OrderItemQueryDto>> orderItemMap = orderItems.stream()
                .collect(groupingBy(OrderItemQueryDto::getOrderId));
        return orderItemMap;
    }

    private List<Long> toOrderIds(List<OrderQueryDto> result) {
        List<Long> orderIds = result.stream()
                .map(OrderQueryDto::getOrderId)
                .collect(toList());
        return orderIds;
    }

    
```

> stream 너무 좋다... 공부 해두길 잘했다..

+ stream 을 적극 활용해서 query문을 최적화하는 방법이다.

+ 우선 version4 와 같이 `toOne` 관계를 조회 하고 , id값을 받아오는데 여기서는 List로 한번에 받아온다. (`toOrderIds()`)

+ 그다음 batch 처럼 직접 in쿼리문을 작성한다!  

+ 결과 값은 스트림 groupingBy로  Id 값으로 묶어버리고 맵으로 만든다.

+ 마지막으로, Orderid 를 키값으로 set해준다. 

+ version4와 흐름은 비슷한데 version5에서는 최대한 묶음으로 처리하여 최적화하고 있다.

+ 결과적으로 쿼리문은 1+1 이 나가게 된다.

<br>



## 4-3 [version6] JPA에서 DTO로 직접 조회 , 플랫데이터 최적화

```java
@GetMapping("/api/v6/orders")
public List<OrderQueryDto> ordersV6() {

    List<OrderFlatDto> flats = orderQueryRepository.findAllByDto_flat();
    return flats.stream()
            .collect(groupingBy(o -> new OrderQueryDto(o.getOrderId(),
                            o.getName(), o.getOrderDate(), o.getOrderStatus(), o.getAddress()),
                    mapping(o -> new OrderItemQueryDto(o.getOrderId(),
                            o.getItemName(), o.getOrderPrice(), o.getCount()), toList())
            )).entrySet().stream()
            .map(e -> new OrderQueryDto(e.getKey().getOrderId(),
                    e.getKey().getName(), e.getKey().getOrderDate(), e.getKey().getOrderStatus(),
                    e.getKey().getAddress(), e.getValue()))
            .collect(toList());

}
```

<img src="https://pbs.twimg.com/profile_images/1595807020044947456/Id4Tqc3O_400x400.jpg" title="" alt="띠용 (@jeonghy79609670) / Twitter" width="189"> 예 ?.....

자세히 보니 리팩터링이 될것같다.

```java
.map(e -> new OrderQueryDto(e.getKey().getOrderId(),
                    e.getKey().getName(), e.getKey().getOrderDate(), e.getKey().getOrderStatus(),
                    e.getKey().getAddress(), e.getValue()))

 // 리팩터링 
.map(e -> {
             e.getKey().setOrderItems(e.getValue());
             return e.getKey();
    })
```



#### OrderQueryRepository

```java
public List<OrderFlatDto> findAllByDto_flat() {
    return em.createQuery(
            "select new" +
                    " jpabook.jpashop.repository.order.query.OrderFlatDto(o.id,m.name,o.orderDate,o.status, d.address, i.name,oi.orderPrice,oi.count)" +
                    " from Order o " +
                    " join o.member m" +
                    " join o.delivery d" +
                    " join o.orderItems oi" +
                    " join oi.item i", OrderFlatDto.class
    ).getResultList();
```



#### OrderFlatDto

```java
@Data
public class OrderFlatDto {

    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;

    private String itemName;
    private int orderPrice;
    private int count;

    public OrderFlatDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, String itemName, int orderPrice, int count) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
        this.itemName = itemName;
        this.orderPrice = orderPrice;
        this.count = count;
    }
}

```



+ 쿼리는 한번이 나간다. 페치 조인했을때를 생각해보자 거기서 select하는 컬럼만 직접 명시 한것이랑 다를 바가 없다. 즉, 결과는 데이터 뻥튀기가 나올것이고, 페이징처리는 당연히 불가능 할 것이다.

+ 또한 컨트롤러 메서드를 보면 직접 stream 을 활용해서 중복 제거를 하고 있다. 

+ 장점이라곤 쿼리한번 나가는 것 밖에 없다.





## 그럼 뭘 써야해?

권장순서

1. 엔티티 조회 방식으로 접근
   
   1. 페치조인으로 쿼리수를 최적화하기 (`xxxToOne`)
   
   2. 컬렉션 최적화
      
      1. 페이징 필요할시 `batchSize` 로 최적화
      
      2. 필요없다. -> 페치 조인 사용

2. 엔티티 방식으로 안되겠다 -> DTO 조회 방식 사용

3. 그래도 안된다 -> `NativeSQL` oR 스프링 JdbcTemplate



> DTO 직접 조회 방식은 성능 최적화나 방식을 변경할때 수많은 코드를 변경해야한다.



> 개발자는 성능 최적화와 코드 복잡도 사이에서 항상 줄타기를 해야한다.
> 
> 복잡한 코드는 성능최적화를 불러온다.
> 
> 그러나 너무 복잡한 코드는 유지보수가 어렵다.
> 
> 음.,.



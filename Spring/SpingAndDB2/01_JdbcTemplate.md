[인프런 김영한님 스프링 DB part2](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-db-2/dashboard)

# 01 데이터 접근기술 -  스프링 JdbcTemplate

## 01-1 소개

#### 장점

+ 설정이 편리하다.
  
  + `Spring-Jdbc` 라이브러리에 포함되어 있다. 스프링에서 JDBC를 사용할때 쓰는 라이브러리로 별도의 설정없이 바로 사용이 가능하다.

+ 반복의 제거
  
  + 이름에서도 알 수 있듯, 템플릿 - 콜백 패턴을 사용해서 대부분의 반복을 대신 처리한다. 이는 1편에서 살펴보았다.
  
  + 앞에서 살펴봤지만 구체적으로 어떤 반복을 처리하냐면
    
    + 커넥션 획득
    
    + `statement`를 준비하고 실행 (PreparedStatement같은)
    
    + 커넥션 종료, `statement`,`resultset` 종료 
    
    + 트랜잭션을 다루기 위한 커넥션 동기화 (TransactionManager를 생각하자. 리포지토리에서 커넥션을 얻을때 트랜잭션 동기화 매니저에서 커넥션을 꺼냈다.)
    
    + 예외발생시 스프링예외 변환기 실행
    
    + 등

#### 단점

+ 동적 SQL을 해결하기 어렵다. (단점 1개? 동적 SQL은 매우 많이 쓰인다. 또 동적으로 처리하는 파라미터가 늘어날수록 구현 불가능정도는 기하급수적으로 증가한다. 밑에서 확인)

#### 설정

설정파일(`applicaton.properties`)에 

`implementation 'org.springframework.boot:spring-boot-starter-jdbc'`  를 추가해준다. 이게 끝임.

<br>

<br>

## 01-2 Jdbc Template을 사용한 리포지토리 (version1)

#### 코드 설명

```java
/**
 * JdbcTemplate
 */
@Slf4j
public class JdbcTemplateItemRepository implements ItemRepository {


     private final JdbcTemplate template;

    public JdbcTemplateItemRepository(DataSource dataSource) {
        this.template = new JdbcTemplate(dataSource);
    }
    /*
    ~
    */
}
```

+ `JdbcTemplate`을 빈으로 등록할 수도 있고, `dataSource` 를 의존관계 주입받아서 생성자로 직접 `JdbcTemplate`을 생성 할 수 있는데 관례상 후자를 많이 사용한다.

<br>

```java
    @Override
    public Item save(Item item) {
        String sql = "insert into item(item_name, price, quantity) values (?,?,?)";
        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(connection -> {
            //자동 증가 키
            PreparedStatement ps = connection.prepareStatement(sql, new String[]{"id"});
            ps.setString(1, item.getItemName());
            ps.setInt(2, item.getPrice());
            ps.setInt(3, item.getQuantity());
            return ps;
        }, keyHolder);

        long key = keyHolder.getKey().longValue();
        item.setId(key);

        return item;
    }
```

+ **save()**
  
  + `insert` , `update` , `delete` SQL에는 `template.update()` 를 사용한다.
  
  + PK값은 비워두고 저장하는데 이때, Auto increment 방식을 사용한다.
  
  + `KeyHolder`를 사용해서 Insert문이 나가고 저장된 PK값을 조회할 수 있다.(직접 해본적도 있지만 이걸 순수 JDBC로 처리하려면 코드가 엄청 복잡하다.)
    
    + 아래 더 쉬운 방법이 있다.

<br>

```java
    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        String sql = "update item set item_name = ?, price=?, quantity=?  where id =?";
        template.update(sql,
                updateParam.getItemName(),
                updateParam.getPrice(),
                updateParam.getQuantity(),
                itemId);

    }
```

+ **update()** 
  
  + 파라미터를 순서대로 넘기고 있다.

<br>

```java
    @Override
    public Optional<Item> findById(Long id) {
        String sql = "select id, item_name,price, quantity from Item where id = ?";

        try {
            Item item = template.queryForObject(sql, itemRowMapper(), id);
            //Result Set 값이 없으면  EmptyResultDataAccessException 던짐

            return Optional.of(item);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }

    private RowMapper<Item> itemRowMapper() {
        return ((rs, rowNum) -> {
            Item item = new Item();
            item.setId(rs.getLong("id"));
            item.setItemName(rs.getString("item_name"));
            item.setPrice(rs.getInt("price"));
            item.setQuantity(rs.getInt("quantity"));
            return item;
        });
    }
```

+ **findById()**
  
  + 데이터 하나 조회시에는 `template.queryForObject()` 를 쓴다.
  
  + 조회 결과가 없을때에는 스프링 예외 변환기에의해 스프링이 정의해놓은 예외인 `EmptyResultDataAccessException` 을 던진다.

+ 반환타입이 객체인경우 함수형 인터페이스 `RowMapper`를 파라미터로 넘겨줘야한다. 즉, 객체를 만드는 방법을 넘겨줘야한다.

<br>

```java
    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        String sql = "select id, item_name,price, quantity from Item";

        //동적 쿼리
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            sql += " where";
        }
        boolean andFlag = false;
        List<Object> param = new ArrayList<>();
        if (StringUtils.hasText(itemName)) {
            sql += " item_name like concat('%',?,'%')";
            param.add(itemName);
            andFlag = true;
        }
        if (maxPrice != null) {
            if (andFlag) {
                sql += " and";
            }
            sql += " price <= ?";
            param.add(maxPrice);
        }
        log.info("sql={}", sql);
        return template.query(sql, itemRowMapper(), param.toArray());
    }
```

+ **findAll()**
  
  + 데이터 접근기술에서는 항상 이부분이 관건인것같다. 바로 동적쿼리에 관한 부분이다.    
  
  + 검색 조건인 `ItemSearchCond` 를 넘기는데 검색조건으로 이름과 가격이있고, 둘 다 넘어올수도있고, 넘어오지 않을수도 있어서 동적으로 처리해야한다.
  
  + 즉, 4가지 상황에 따라 동적으로 처리하기위해 sql을 조건 마다 결합하는 식으로 처리했다.

<br>

<br>

## 01-3 NamedParameterJdbcTemplate을 사용한 리포지토리 (version2)

+ version1에서는 파라미터를 넘길때 순서대로 넘겼다. 그러나 여기에는 치명적인 단점이 있다. 누군가 그 의도적이던 실수이던 순서를 바꾸거나해서 값이 잘못들어가버리면 복구는 걷잡을수 없어진다. 

> 모호한 코드는 버리자. 조금 복잡하더라도 명확한 코드를 지향하자.

```java
@Slf4j
public class JdbcTemplateItemRepositoryV2 implements ItemRepository {


    //private final JdbcTemplate template;
    private final NamedParameterJdbcTemplate template;

    public JdbcTemplateItemRepositoryV2(DataSource dataSource) {
        this.template = new NamedParameterJdbcTemplate(dataSource);
    }
}
```

+ 이름 파라미터를 사용하기위해 `JdbcTemplate`이 아니라 `NamedParameterJdbcTemplate`을 사용한다.

<br>

```java
    @Override
    public Item save(Item item) {
        String sql = "insert into item(item_name, price, quantity) " +
                "values (:itemName,:price,:quantity)";

        SqlParameterSource param = new BeanPropertySqlParameterSource(item);

        KeyHolder keyHolder = new GeneratedKeyHolder();

        template.update(sql, param, keyHolder);

        long key = keyHolder.getKey().longValue();
        item.setId(key);

        return item;
    }
```

+ save()
  
  + 이제는 ? 가 아니라 파라미터이름 을 `:itemName` 을 지정해서 넘겨준다.
  
  + 파라미터를 넘기는 방법에는 여러가지가 있는데, 여기서는 `BeanPropertySqlParameterSource` 를 사용한다. 자바빈 프로퍼티 규약으로 객체를 넘겨주면 자동으로 파라미터 객체를 생성해준다. 매우 편리한 방법이다. 파라미터 이름을 객체 필드명과 맞춰 주면 된다. 여기선 문제가 없지만 select문을 쓸때 필드랑 컬럼이랑 일치하지않으면 사용할 수 없다.
  
  + 또, 생성 키 또한 쉽게 조회할 수 있다 

<br>

```java
    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        String sql = "update item set item_name = :itemName, price= :price, quantity= :quantity " +
                "where id = :id";


        SqlParameterSource param = new MapSqlParameterSource()
                .addValue("itemName", updateParam.getItemName())
                .addValue("price", updateParam.getPrice())
                .addValue("quantity", updateParam.getQuantity())
                .addValue("id", itemId);
        template.update(sql, param);

    }
```

+ update문 에서는 `MapSqlParameterSource` 을 사용했다. Dto에는 id가 없기 때문이다.



<br>

```java
private RowMapper<Item> itemRowMapper() {
        return BeanPropertyRowMapper.newInstance(Item.class); //camel case 변환 지원
    }
```

+ RowMapper 도 더 쉽게 얻을 수 있다. ResultSet과 객체의 필드명을 비교해서 데이터를 변환해준다. 
  
  + CamelCase도 지원하고 언더스코어도 지원한다. 근데 만약 아예 다른 이름이라면? 별칭을 쓰면된다.



<br>

<br>

## 01-4 SimpleJdbcInsert 사용 (version3)

+ 인서트문 작성하기 귀찮아 할때 쓴다.

```java
public class JdbcTemplateItemRepositoryV3 implements ItemRepository {


    private final NamedParameterJdbcTemplate template;
    private final SimpleJdbcInsert jdbcInsert;

    public JdbcTemplateItemRepositoryV3(DataSource dataSource) {
        this.template = new NamedParameterJdbcTemplate(dataSource);
        this.jdbcInsert = new SimpleJdbcInsert(dataSource)
                .withTableName("item")
                .usingGeneratedKeyColumns("id");
                //.usingColumns("item_name","quantity",price) 생략가능
    }
}
```

+ 인서트문이 나가는 테이블을 지정해줘야하고, pk컬럼명을 지정해야한다.

+ 이렇게하면 컴파일 될때 Insert쿼리문을 만들어준다.



<br>

```java
    @Override
    public Item save(Item item) {
        SqlParameterSource param = new BeanPropertySqlParameterSource(item);
        Number key = jdbcInsert.executeAndReturnKey(param);
        item.setId(key.longValue());

        return item;
    }
```

+ 파라미터만 넘겨주면 알아서 인서트문 날려주고 Pk 를 반환한다.

+ 인서트문을 획기적으로 줄일 수 있다.







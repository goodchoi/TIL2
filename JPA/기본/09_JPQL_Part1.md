[인프런 김영한님 강의 - 자바 ORM 표준 JPA 프로그래밍 -기본편](https://www.inflearn.com/course/ORM-JPA-Basic)

# 09. 객체지향 쿼리언어(JPQL)

JPA는 다양한 쿼리 방법을 지원한다.

+ **JPQL**

+ JPA Criteria

+ **QueyDSL**

+ 네이티브 SQL

+ JDBC API 직접사용, MyBatis, SpringJdbcTemplate 함께 사용하기

## 09-1 JPQL(Java Persistence Query Language)

### JPQL 이란?

+ 회원을 한명 조회하려면 `EntityManager.find()`를 해야할 것이다. 근데 만약 나이가 18살 이상인 회원을 모두 검색 하고 싶다면 ?

+ JPA는 SQL을 추상화한 JPQL 이라는 객체 지향 쿼리언어를 제공한다.(추상화 했기때문에 특정 데이터베이스 SQL에 의존하지 않는다.)

+ SQL과 문법이 유사하다.

+ 하지만 JPQL은 테이블이 아닌 **엔티티 객체를 대상으로 쿼리한다.**

+ JPQL은 결국 SQL로 변환된다.

### JPQL과 실행된 SQL

```java
String jpql = "select m from Member m where m.age > 18";
List<Member> result = em.createQuery(jpql, Member.class)
        .getResultList();
```

 

```java
/*
실행된 SQL select
          m.id as id,
          m.age as age,
          m.USERNAME as USERNAME,
          m.TEAM_ID as TEAM_ID
          from
          Member m
          where
          m.age>18
*/
```

## JPQL 문법

+ 엔티티와 속성은 대소문자 구분해야한다.

+ 키워드는 구분하지 않아도 된다.

+ 테이블 이름이 아닌 엔티티 이름을 사용해야한다

+ **별칭은 필수이다.**

### TypedQuery, Query

+ TypedQuery : 반환 타입이 명확할때 사용한다.

```java
TypedQuery<Member> query =
    em.createQuery("SELECT m FROM Member m", Member.class);
```

+ Query : 반환 타입이 명확하지 않을때 사용한다.

```java
Query query =
    em.createQuery("SELECT m.username, m.age from Member m");
```

### 결과 조회 API

+ `query.getResultList()` : <u>결과가 하나 이상일떄</u> , 리스트를 반환한다.
  
  + 결과가 없으면 빈 리스틀 반환한다.

+ `query.getSingleResult()` : <u>결과가 정확히 하나 일때,</u> 단일 객체를 반환한다.
  
  + 결과가 없거나 둘이상이면 예외를 반환한다. -> 골치아프게 된다. SpringData JPA에서는 그래서 예외처리를 해준다고한다.

### 파리미터 바인딩

+ 이름 기준이있고, 위치기준이있는데 위치 기준을 사용한는것은 바람직하지 않다.

```java
query = "SELECT m FROM Member m where m.username=:username"
query.setParameter("username", usernameParam);
```

### 프로젝션

+ `SELECT` 절에 조회할 대상을 지정하는 것을 말함.

+ `SELECT m FROM Member m` -> 엔티티 프로젝션 

+ `SELECT m.team FROM Member m` -> 엔티티 프로젝션
  
  + 엔티티 프로젝션으로 얻은 결과값들은 모두 영속성 컨텍스트에 의해 관리된다.

+ `SELECT m.address FROM Member m` -> 임베디드 타입 프로젝션

+ `SELECT m.username, m.age FROM Member m` -> 스칼라 타입 프로젝션

### 프로젝션 조회

+ Query타입으로 조회
  
  + 그냥 반환 Class 타입 명시 안하고 받아서 처리하는 방법.

+ Object[] 타입으로 조회
  
  + 여러값들을 조회 할때 Object[]로 받겠다는말 즉 `getResultList()`를 한다면 반환타입은 마치` List<Object[]> ` 로받고 `Object[]` 안에 값들이 순서대로 담기게됨.

+ new 명령어로 조회
  
  + 단순 값을 DTO로 조회 할때 사용
  
  ```java
   List<MemberDTO> resultList = em.createQuery("select new jpql.MemberDTO(m.username,m.age) from Member m"
               , MemberDTO.class)
               .getResultList();
      
  ```
  
  + 쿼리문안에 DTO의 패키지 명을 포함한 전체 클래스명을 입력해주어야한다.(QueryDsl사용시 패키지명을 따로 Import(?) 해서 편하게 사용하긴한다.)

+ 조회하는 값들과 순서와 타입이 일치하는 생성자가 필요하다.

### 페이징 API

+ 쿼리문으로 페이징 처리를 할때의 이루 말할 수없는 고뇌의 시간을 떠올려보자

+ JPA는 페이징 처리를 지원한다.

+ `setFirstResult(int startPosition)` : 조회 시작위치 -> 0부터 시작

+ `setMaxResults(int maxResult)` : 조회할 데이터 수
  
  ```java
  String jpql = "select m from Member m order by m.name desc"; 
  List<Member> resultList = em.createQuery(jpql, Member.class)
        .setFirstResult(10)
        .setMaxResults(20)
        .getResultList();
  ```

+ 설정한 데이터베이스 방언에 맞게 쿼리문 이 동작한다.

### 조인

+ 내부 조인 :
  
  `SELECT m FROM Member m [INNER] JOIN m.team t`

+ 외부 조인 :
  
  `SELECT m FROM Member m LEFT [OUTER] JOIN m.team t`

+ 세타 조인 :
  
  `select count(m) from Member m, Team t where m.username = t.name`
  
  + SQL에서 Join 할때랑 다르게 on 절을 따로 명시하지 않았다.
  + 대신 JOIN을 누구랑 하는지 잘 볼 필요가 있다**<mark>. FROM 의 엔티티 내부의 연관관계 필드와 JOIN하고 있다.</mark>**

+ On절 활용 
  
  + 조인대상 필터링 할떄
    
    + `SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'A'`
    
    + 필터링 할 조건을 on절에 명시하면됨
  
  + 연관관계 없는 엔티티 외부 조인 (아무 상관없는 놈들끼리 값만 비교할경우)
    
    + `SELECT m, t FROM  
      Member m LEFT JOIN Team t on m.username = t.name`
    
    + 이떄는 왜 on절 적어줘야하냐고? 연관관계가 없으니깐요.. 
    
    + <img title="" src="https://mblogthumb-phinf.pstatic.net/20160504_208/ttoongmoong1105_1462337383624Q9zCd_JPEG/%C1%B6%BC%BC%C8%A3%BF%F4%B0%DC%C1%D7%B0%DA%B3%D70006.jpg?type=w800" alt="조세호 패러디 : 조세호 왜 안왔어요 ? : 네이버 블로그" width="355">

### 서브쿼리

예제로 알아보자.

+ 팀 A 소속인 회원 (회원-팀 다대일 양방향관계)
  
  + `select m from Member m  where exists (select t from m.team t where t.name = ‘팀A')` -> EXISTS : 서브쿼리에 값이 존재하면 참

+ 전체 상품 각각의 재고보다 주문량이 많은 주문들
  
  + `select o from Order o  where o.orderAmount > ALL (select p.stockAmount from Product p)`  -> ALL : 모두 만족하면 참

+ 어떤 팀이든 팀에 소속된 회원
  
  + `select m from Member m  where m.team = ANY (select t from Team t)` -> ANY,SOME = 조건을 하나라도 만족하면 참.

+ 한계
  
  + JPA에서는 WHERE, HAVING 절에서만 사용가능하다.
  
  + SELECT절에서도 사용가능(하이버네이트)
  
  + **<u>하지만...FROM절에서 서브쿼리는 불가능하다.</u>**
    
    + 조인으로 해결할수 있는경우가 대다수 이기때문에 조인으로 어떻게든 풀어내야한다.
    
    + ~~? 하이버네이트 6에서는 지원한다고한다.? 그렇다고한다.~~

### 그외 지원

+ CASE 식 , COALESCE, NULLIF

+ 기본함수(CONCAT,SUBSTRING,TRIM ,lOCATE등)
  
  + 필요하면 그때 찾아서 쓰자. 외울거냐?

### 사용자 정의 함수 호출하기

+ 방언에 추가하는 식으로 쓰인다.
  
  + DB방언을 상속 받고, 등록한다.

```java
package dialect;

import org.hibernate.dialect.H2Dialect;
import org.hibernate.dialect.function.StandardSQLFunction;
import org.hibernate.type.StandardBasicTypeTemplate;
import org.hibernate.type.StandardBasicTypes;

public class MyH2dialect extends H2Dialect {

    public MyH2dialect() {
        registerFunction("blah_blah",new StandardSQLFunction("name", StandardBasicTypes.STRING);
    }

}

```

+ 헷갈리면 `H2Dialect` 를 들어가보자 위와 비슷하게 다 등록이 되어있다.

+ 삽질 몇번만하면 바로 쓸수 있을듯.

+ 사용하기
  
  ```java
  select function('group_concat', i.name) from Item i
  ```

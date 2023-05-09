[인프런 김영한님 강의 - 자바 ORM 표준 JPA 프로그래밍 -기본편](https://www.inflearn.com/course/ORM-JPA-Basic)

# 10. 객체지향 쿼리언어 (JPQL) - part2

## 10-1 경로 표현식

:JPQL 안에서 `m.username` 처럼 점을 찍어 객체 그래프를 탐색하는 것을 말함.

### 용어 정리

+ 상태 필드 : `m.username` 처럼 단순히 값을 저장하기 위한 필드

+ 연관 필드 : 연관관계를 위한필드
  
  + 단일 값 연관 필드 : `@ManyToOne`, `@OneToOne` , 대상이 엔티티 
    
    + `m.team`
  
  + 컬렉션 값 연관 필드 : `@OneToMany` , `@ManyToMany` , 대상이 컬렉션인 경우 
    
    + `m.orders`

+ 묵시적 조인 : 객체 그래프 탐색시 내부적으로 inner join이 발생하는 경우
  
  + 묵시적 조인은 직관적이지않고 의도치 않게 join문이 나가므로 **쿼리 튜닝이 힘들다.**
  
  + 그래서 sql문과 비슷하게 join을 직접 명시하던지 하는것이 좋다.

### 특징

+ **상태필드**의 경우 경로의 끝이므로 탐색이 더이상 불가능하다.

+ **단일값 연관경로**의 경우 묵시적 내부 조인이 발생한다. 탐색이 가능하다.

```java
List<Team> resultList = em.createQuery("select m.team from Member m", Team.class)
                    .getResultList();

/* 실제 쿼리문
        select
            team1_.TEAM_ID as team_id1_3_,
            team1_.name as name2_3_ 
        from
            Member member0_ 
        inner join
            Team team1_ 
                on member0_.TEAM_ID=team1_.TEAM_ID
*/
```

+ **컬렉션 값 연관경로**의 경우 묵시적 내부 조인이 발생하나,탐색이 불가능하다.
  
  + FROM절에서 명시적 조인을 통해 별칭을 얻으면 탐색이 가능.

### 명시적 조인, 묵시적 조인

+ 둘의 실제 sql은 같다.
  
  ```java
  select m.team from Member m //묵시적조인
  
  select m from Member m join m.team t // 명시적 조인
  
  select
      team1_.TEAM_ID as team_id1_3_,
      team1_.name as name2_3_ 
  from
      Member member0_ 
  inner join
      Team team1_ 
      on member0_.TEAM_ID=team1_.TEAM_ID
  ```

+ **가급적 명시적 조인을 사용하라**

+ 묵시적조인은 조인이 일어나는 상황을 한눈에 파악하기 어렵다.

<br>

---

## :exclamation: 매우 중요  :exclamation:

## 10-2 페치 조인 (Fetch Join)

> Fetch 조인을 모르고는 실무 못한다. 성능과도 직결되기 때문에 매우 중요하다!
> 
> 특히 Join은 쿼리 성능에 아주 지대한 영향을 미친다. 
> 
> 이떄 fetch Join을 적절히 사용해서 성능을 최대로 튜닝해야한다.

### 페치 조인이란

+ SQL의 조인의 종류가 아니다.

+ **JPQL에서 성능 최적화**를 위해 제공되는 기능이다.

+ **연관된 엔티티나 컬렉션을 SQL한번에 **조회하는 기능이다.

### 엔티티 페치 조인

+ 회원을 조회하면서 연관된 팀도 함께 조회하려한다. **둘 다 필요한 상황**

```java
String jpql = "select m from Member m join fetch m.team"; 
List<Member> members = em.createQuery(jpql, Member.class)
                    .getResultList(); 

for (Member member : members) {
//페치 조인으로 회원과 팀을 함께 조회해서 지연 로딩X 
System.out.println("username = " + member.getUsername() + ", " +
}

/* 실제 쿼리
         select
            member0_.id as id1_0_0_,
            team1_.TEAM_ID as team_id1_3_1_,
            member0_.age as age2_0_0_,
            member0_.TEAM_ID as team_id5_0_0_,
            member0_.type as type3_0_0_,
            member0_.username as username4_0_0_,
            team1_.name as name2_3_1_ 
        from
            Member member0_ 
        inner join
            Team team1_ 
                on member0_.TEAM_ID=team1_.TEAM_ID
*/
```

### 컬렉션 페치 조인

+ 일대다 관계에서 컬렉션에 페치조인을 쓸 수 있다.

```java
String jpql = "select t from Team t join fetch t.members where t.name = 'teamA'";

List<Team> teams = em.createQuery(jpql, Team.class).getResultList();
for(Team te : teams) {
    System.out.println("teamname = " + te.getName() + ", team = " + te);
    te.getMembers().forEach(System.out::print);
    System.out.println();
}

/*
            select
            team0_.TEAM_ID as team_id1_3_0_,
            members1_.id as id1_0_1_,
            team0_.name as name2_3_0_,
            members1_.age as age2_0_1_,
            members1_.TEAM_ID as team_id5_0_1_,
            members1_.type as type3_0_1_,
            members1_.username as username4_0_1_,
            members1_.TEAM_ID as team_id5_0_0__,
            members1_.id as id1_0_0__ 
        from
            Team team0_ 
        inner join
            Member members1_ 
                on team0_.TEAM_ID=members1_.TEAM_ID 
        where
            team0_.name='teamA'


*/
```

+ 슬슬 join의 압박이 느껴지지만 쫄거 없다. 테이블 8개 9개 join하던 그때를 떠올려라 이정도는 너무쉬워서 눈물이 날 지경이다.

+ 결과는 어떻게 나올까?
  
  ```java
  teamname = teamA, team = jpql.Team@24097e9b
  Member{id=4, username='회원1', age=10} Member{id=5, username='회원2', age=10}
  teamname = teamA, team = jpql.Team@24097e9b
  Member{id=4, username='회원1', age=10} Member{id=5, username='회원2', age=10}
  ```
  
  + 보이는가? 정확히 같은 결과가 두번 출력되었다. 즉데이터가 뻥튀기 되었다. 이건 왜이럴까? 일대다 의 연관관계이기 때문이다. 즉, DB는 컬렉션의 개념이 없다. 영속성 컨텍스트에는 중복없이 들어가겠지만 List로 받은 결과 값은 중복이 있을 수밖에 없다. 5초만 생각해보면 당연하것이다.
    
    + distinct조건을 주면된다. distinct는 SQL에도 들어가고 어플리케이션에서도 작동하게 되는데 위의 경우 SQL에서는 어차피 효용이없다. -> 당연
    
    + 어플리케이션에서 실제로 중복을 제거하는 작동을 한다.
    
    + 하이버네이트6 에서는 distinct없이도 중복을 제거해준다고 한다. ..... 뭐임..
  
  + 다대일의 경우 이런 걱정이없다. 왜냐고? 다대일이니깐요..컬렉션이 없잖아요..

+ **중요한점은 페치조인으로 N+1 문제를 해결 했다는 점이다.**

### 페치조인과 일반 조인의 차이

+ JPQL은 결과를 반환할때 연관관계를 고려하지않는다. 그냥 SELECT절에 지정한 엔티티만 긁어온다.

+ 페치조인은 연관된 엔티티를 함께 조회한다는게 의미가 있다.

> (추가 ) 이부분에서 설명이 부족해서 조금 헤매였는데 정확히는 이 차이가있다.
> 
> 일반조인의 경우 조회하는 주체가 되는 Entity만 영속화한다
> 
> fetch join은 연관된 entity를 모두 영속화한다. (명시하지않아도)

### 한계

+ 페치 조인 대상에는 별칭을 줄 수 없다. ->하이버네이트는 가능, 하지만 가급적 사용 x

+ 둘 이상의 컬렉션은  페치조인 할 수없다. -> 데이터 뻥튀기만 생각해도,,

+ 컬렉션 페치조인은 페이징 처리를 할 수없다.

+ 모든 것을 페치 조인으로 해결 할 수는 없다.

+ 여러 테이블을 조인해서 엔티티가 가진 모양이 아닌 전혀 다른 결과를 내야한다. -> **일반조인 + DTO 객체**

### Batch

`@BatchSize(size= 100) ` 이런식으로 줄 수 있는데 이 아이는 in 쿼리문을 사용한다.

```java
String jpql = "select t from Team t";
List<Team> resultList = em.createQuery(jpql, Team.class).setFirstResult(0).setMaxResults(3).getResultList();

System.out.println("resultList[0].get = " + resultList.get(0).getMembers());
```

+ 컬렉션 에 페이징처리를 꼭하고싶다 ->BatchSize를 설정하면된다.

+ 이건 기본 글로벌 로딩전략이 우선인듯? (실제사용시에 호출됨 )

실제 쿼리문

```sql
/* load one-to-many jpql.Team.members */ select
        members0_.TEAM_ID as team_id5_0_1_,
        members0_.id as id1_0_1_,
        members0_.id as id1_0_0_,
        members0_.age as age2_0_0_,
        members0_.TEAM_ID as team_id5_0_0_,
        members0_.type as type3_0_0_,
        members0_.username as username4_0_0_ 
    from
        Member members0_ 
    where
        members0_.TEAM_ID in (
            ?, ?, ?
        )
```

+ in에 담기는 값들에 주의하라. 기본 조회 쿼리에서 영속성 컨텍스트에 등록된 teamId 들이 담길테고 이때 batch size에 맞게 들어간다. 굳

<br>

## 10-3 Named 쿼리 - 정적쿼리

+ 미리 쿼리를 등록해놓고 사용할 수있다.

+ 어노테이션 혹은 XML에 정의할 수 있다.

+ **가장 큰 장점은 애플리케이션 로딩 시점에 쿼리를 검증 할 수있다는것!**

```java
@Entity
@NamedQuery(
        name = "Member.findByUsername",
        query = "select m from Member m where m.username = :username"
)
public class Member {
}
/////////////////////////

////사
List<Member> resultList =
em.createNamedQuery("Member.findByUsername", Member.class)
.setParameter("username", "회원1")
.getResultList();
```

+ SprinData JPA에서 유용하게 쓰이는것같다.

<br>

## 10-4 벌크연산

+ 일관변경을 하고 싶으면 어떻게?

```java
String qlString = "update Product p " +
                  "set p.price = p.price * 1.1 " +
                  "where p.stockAmount < :stockAmount";
int resultCount = em.createQuery(qlString)
                    .setParameter("stockAmount", 10)
                    .executeUpdate();
```

+ 이때, 벌크연산은 영속성 컨텍스트 무시하고 DB에 바로 SQL날리기떄문에 시점을 잘정해야한다.
  
  + 벌크연산을 먼저 수행하는 방법
  
  + **벌크 연산 수행후 영속성 컨텍스트를 초기화 하는 방법**

<br>

## -기본편 끝

재밌네.. 잘 알고 써야지 막 쓰면 난리나겠는데? 

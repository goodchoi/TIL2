[인프런 김영한님 강의 - 스프링 DATA JPA](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%EC%8B%A4%EC%A0%84/dashboard)

# 01 - 쿼리 메소드 기능

 스프링 데이터 JPA는 인터페이스에 자주 쓰이는 메소드들을 모두 정의 해놓았다. 여기에 더 나아가서 여러가지 조건에따라 쿼리문을 다르게 날리고 싶을때 쓸 수 있는 편리한 기능까지 제공한다.

#### 쿼리 메소드 기능 3가지

+ 메소드 이름으로 쿼리를 생성

+ 이름으로 JPA Named Query 호출

+ `@Query` 어노테이션을 사용해서 리포지토리 인터페이스에 쿼리 직접 정의

실제적인 사용성은 1번과 3번이 적합한데, 조건이 적으면 1번이 가장 편하고 그외에는 3번을 쓰는게 좋다.

## 01-1 메소드 이름으로 쿼리 생성

#### 순수 JPA

```java
public List<Member> findByUsernameAndAgeGreaterThan(String username,int age) {
        return em.createQuery("select m from Member m where m.username =:username and m.age >=:age",Member.class)
                .setParameter("username",username)
                .setParameter("age",age)
                .getResultList();
    }
```

#### 메소드 이름으로 쿼리 생성(스프링 데이터 JPA)

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
}
```

<img title="" src="https://media3.giphy.com/media/ghuvaCOI6GOoTX0RmH/giphy.gif?cid=ecf05e47xij4dp1b9l4fqt45q3dvfy92pri7x6np10mq23tk&ep=v1_gifs_search&rid=giphy.gif&ct=g" alt="The Office gif. Steve Carell as Michael Scott purses his lips and raises his eyebrows in annoyance as he says, " data-align="center" width="237">

메서드의 이름을 엔티티의 필드명을 이용하여 저들이 정의해놓은 관례에 맞게만 작성하면 알아서 메서드드를 만들어준다.. 

큰 단점은 조건이 세개이상이 추가될수록 메서드이름이 지나치게 길어진다는 점이다.. 아주 간단한 메서드 에 쓰면 상당히 좋을 것 같다.

<br>

## 01-2 JPA NamedQuery

JPA의 NamedQuery를 호출 할 수 있다.

#### 엔티티에 NamedQuery 정의하기

```java
@Entity
@NamedQuery(
     name="Member.findByUsername",
     query="select m from Member m where m.username = :username")
public class Member {
 /////
}
```

<br>

#### JPA에서 직접 NamedQuery 호출

```java
public class MemberRepository {

     public List<Member> findByUsername(String username) {
         ...
         List<Member> resultList =
             em.createNamedQuery("Member.findByUsername", Member.class)
                 .setParameter("username", username)
                 .getResultList();
     }
}
```

<br>

#### 스프링 데이터 JPA로 NamedQuery 사용하기.

```java
@Query(name = "Member.findByUsername")
List<Member> findByUsername(@Param("username") String username);
```

딱 봐도 사용성이 그렇게 좋아보이진 않는다. 이런게 있구나 하고 넘어가자

<br>

## 01-3 @Query , 리포지토리에 쿼리 직접 정의하기

```java
import org.springframework.data.jpa.repository.*;


public interface MemberRepository extends JpaRepository<Member, Long> {

    @Query("select m from Member m where m.username = :username and m.age = :age")
    List<Member> findUser(@Param("username") String username, @Param("age") int age);
}
```

`NamedQuery` 와 `@Query` 를 썼을때 가장 좋은 점은 애플리케이션 실행 시점에 문법오류를 발견할 수 있다는 점이다.

<br>

## 01-4 @Query로 DTO 조회하기

```java
@Query("select new study.datajpa.dto.MemberDto(m.id,m.username,t.name)from Member m join m.team t")
List<MemberDto> findMemberDto();
```

기본적으로 JPA 와 사용방식이 동일하다. 

#### 파라미터 바인딩

JPA와 마찬가지로 위치기반 , 이름기반 둘다 지원하지만 이름기반으로 사용하는 것이 가장 안전하다.

사용방식은 하나만 알면된다 위에서도 확인 할 수 있듯 JPQL에 파라미터를 넣고 메서드 파라미터에 `@Param` 으로 명시해주면 된다.

이런 것도 당연히 가능하다. in 쿼리를 사용하고 파라미터로 `컬렉션` 을 넘긴다.

```java
@Query("select m from Member m where m.username in :names")
List<Member> findByNames(@Param("names") List<String> names);
```

<br>

## 01-5 반환타입

반환타입도 기가막히다. 단건으로 리턴타입을 명시해놓으면 `getSingleResult()` 를 호출하고 컬렉션이면 `getResultList` 를 호출한다. 

[Spring Data JPA - Reference Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repository-query-return-types)에서 확인할 수 있지만 다양한 리턴타입을 제공한다.

void, 원시타입, Optional, Stream , Collection ,List 등 다양하게 있다.

<br>

## 01-6 페이징과 정렬

#### 순수 JPA의 페이징

```java
public List<Member> findByPage(int age, int offset, int limit) {
        return em.createQuery("select m from Member m where m.age = :age order by m.username desc",Member.class)
                .setParameter("age",age)
                .setFirstResult(offset)
                .setMaxResults(limit)
                .getResultList();
    }

    public long totalCount(int age) {
        return em.createQuery("select count(m) from Member m where m.age = :age", Long.class)
                .setParameter("age",age)
                .getSingleResult();
    } //페이징을 하기위해선 전체 카운트수가 필요하다.
```

<br>

##### 스프링 데이터 JPA 를 사용한 페이징

```java
Page<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용
Slice<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용안함
List<Member> findByUsername(String name, Pageable pageable); //count 쿼리 사용안함
List<Member> findByUsername(String name, Sort sort);
```

페이징과 정렬 파라미터

+ `org.springframework.data.domain.Sort` : 정렬 정보를 가진 객체 - 클래스임

+ `org.springframework.data.domain.Pageable`  : 페이징 기능 - 인터페이스 , 내부에 `Sort`를 포함 하고 있음

반환 타입

+ `Page` : 반환 결과와 전체 카운트 결과를 포함하는 객체

+ `Slice` : 전체 카운트 결과없이 (쿼리문 안나감) 다음 페이지 확인 할떄
  
  + 내부적으로 limit+1

#### 사용예제(Page)

```java
@Test
void paging() {
    memberRepository.save(new Member("member1" , 10));
    memberRepository.save(new Member("member2" , 10));
    memberRepository.save(new Member("member3" , 10));
    memberRepository.save(new Member("member4" , 10));
    memberRepository.save(new Member("member5" , 10));

    int age= 10;

    PageRequest pageRequest = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC, "username"));
    //0페이지에서 3개 가져와라 //

    Page<Member> page = memberRepository.findByAge(age, pageRequest);

    page.map(member -> new MemberDto(member.getId(), member.getUsername(), null));

    List<Member> content = page.getContent();
    long totalElements = page.getTotalElements();// totalcount

    assertThat(content.size()).isEqualTo(3);
    assertThat(totalElements).isEqualTo(5);
    assertThat(page.getNumber()).isEqualTo(0);
    assertThat(page.getTotalPages()).isEqualTo(2);
    assertThat(page.isFirst()).isTrue();
    assertThat(page.hasNext()).isTrue();
}
```

+ `PageRequest` 는 `Pageable` 의 구현체이다.
  
  + 첫번째 파라미터 : 현재 페이지, 
  
  + 두번째 파라미터 : 조회 할 데이터 수
  
  + 세번쨰 : 정렬조건 ->없어도 된다.

+ `Sort.by` 는 `Sort` 클래스의 팩토리 메서드이다. 

+ `Page` 는 0부터 시작한다.

+ 구글링 해보니 스프링 `argumentResolver` 가 처리하는 파라미터중에 `Pageable` 이 있으므로 메서드 정의만 해놓으면 파라미터로 바로 받아서 요청을 처리하는 그림으로 사용하면 될것같다.

+ Page 방식은 count 쿼리가 계속 나가기 때문에 비용이 크다.



#### 사용예제(Slice)

```java
@Test
void slicing() {
    memberRepository.save(new Member("member1" , 10));
    memberRepository.save(new Member("member2" , 10));
    memberRepository.save(new Member("member3" , 10));
    memberRepository.save(new Member("member4" , 10));
    memberRepository.save(new Member("member5" , 10));

    int age= 10;

    PageRequest pageRequest = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC, "username"));
    //0페이지에서 3개 가져와라

    Slice<Member> page = memberRepository.findByAge(age, pageRequest);

    List<Member> content = page.getContent();

    assertThat(content.size()).isEqualTo(3);
    assertThat(page.getNumber()).isEqualTo(0);
    assertThat(page.isFirst()).isTrue();
    assertThat(page.hasNext()).isTrue();
}
```

+ slice 는 따로 카운트 쿼리가 나가지 않는다.

+ `더보기` 를 생각하자.



>  참고 : count 쿼리는 분리할 수 있다. join 이 발생하는 쿼리문이면 카운트 쿼리도 join문이 나간다. -> 사실 count할때는 조인 쿼리가 필요가 없다. 

```java
@Query(value = “select m from Member m”,
     countQuery = “select count(m.username) from Member m”)
Page<Member> findMemberAllCountBy(Pageable pageable);
```

<br>



## 01-7 벌크성 수정쿼리

```java
@Modifying // executeUpdate 실행 없으면 getResultList 이런거 실행함.
@Query("update Member m set m.age = m.age +1 where  m.age >= :age")
int bulkAgePlus(@Param("age") int age);
```

+ `@Modifying` 을 붙이지 않으면 excuteUpdate를 실행하지않는다.

+ 기본편에서도 다루었지만 벌크 연산을 사용할때는 두가지 방법중 하나를 선택한다.
  
  + 가장 먼저 벌크 연산을 수행한다.
  
  + 벌크연산 직후 영속성 컨텍스트를 초기화한다. 

+ DB와 영속성 컨텍스트가 일치하지 않을 수 있기 때문.



<br>

## @EntityGraph

JPQL없이 페치조인을 사용할 수 있다.

```java
@Query("select m from Member m left join fetch  m.team")
List<Member> findMemberFetch();

//공통 메서드 오버라이드
@Override
@EntityGraph(attributePaths = {"team"})
List<Member> findAll();


//메서드 이름으로 쿼리에서 특히 편리하다.
@EntityGraph(attributePaths = {"team"})
List<Member> findByUsername(String username)
```

<br>



## JPA Hint & Lock

```java
@QueryHints(value = @QueryHint(name = "org.hibernate.readOnly", value =
"true"))
Member findReadOnlyByUsername(String username);
```

- ReadOnly option을 걸 수 있다.

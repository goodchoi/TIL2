[인프런 김영한님 강의 - 자바 ORM 표준 JPA 프로그래밍 -기본편](https://www.inflearn.com/course/ORM-JPA-Basic)



# 7.프록시와 연관관계 관리



## 7-1 프록시 (Proxy)

<img title="" src="IMG/proxy.png" alt="" width="131" data-align="center">

+ 프록시 객체는 실제 엔티티 클래스를 상속받아서 만들어진다.

+ 내부에 실제 객체의 참조(target)를 보관한다.
  
  + 영속 엔티티를 참조하는 역할을 한다.

+ 프록시 객체를 호출하면 실제 객체를 호출한다. 마치 매개체역할

### - 동작 과정

<img title="" src="IMG/proxy_works.png" alt="" width="529" data-align="center">

+ 프록시 객체는 처음 사용시 한번 초기화 되면 끝이다. 즉, 한번 프록시 객체를 썼다면 계속 쓰게 된다는 의미. 갑자기 실제로 엔티티 객체로 바꿔치기 하거나 하는게 아니다.

+ em.getRefence() 를 통해 호출하여 얻을 수 있고, 만약 영속성 컨텍스트에 찾고자하는 엔티티가 이미 존재할시 실제 그 엔티티를 반환한다. -> 프록시 객체를 얻지 않는다.

+ 사실 실무적으로 중요하진 않지만 이 프록시라는 주제에서 중요한 메커님즘은 같은 트랜잭션안에서 똑같은 대상을 참조? 혹은 호출 했을때 항상 같은 값을 불러 와야한다는 것이다
  
  ```java
  try {
              Member member = new Member();
              member.setUsername("테스트");
              em.persist(member);
  
              em.flush();
              em.clear();
  
              Member reference = em.getReference(Member.class, member.getId());
              Member member1 = em.find(Member.class, member.getId());
              
              reference == member1?
  
              tx.commit();
  
        }
  ```
  
  + 위의 트라이문 안에서 refence와 member1은 같을까 ? 같다!
    
    + member1은 Member클래스여야하지만, 위의 메커니즘에 맞게 동작하기위해 프록시 객체를 반환한다... ㄷ ㄷ  순서가 바뀌면 ? reference는 Member클래스일 것이다.

+ 영속성 컨텍스트의 도움을 받을 수 없는 준영속 상태일떄 프록시를 초기화하면 문제가 발생한다.
  
  + `org.hibernate.LazyInitializationException` 예외를 터뜨린다.
  
  + 실무에서 아주 많이 접하게될 예외이다.
  
  + 한 트랜잭션이 끝나거나 영속성 컨텍스트가 닫혔을때 프록시를 호출하거나 초기화하려고하면 예외가 발생.

<br>

## 7-2 지연로딩과 즉시로딩

> 이런 의문점이 들때는 JPA로 어떻게 처리할 수 있을까? 
> 
> Member를 조회하는데 굳이 Team을 무조건 함께 조회해야할까? 
> 
> 필요없는 상황에서도? Mybatis를 사용하면 join을 안하는 쿼리문을 날리면 그만이다.
> 
> JPA에서는 방법을 어떻게 마련해놨을까?

<br>

+ **해결책: 지연 로딩(Lazy)을 사용해서 프록시로 조회한다.**
  
  + 지연로딩을 하면 해당 필드의 테이블과 Join 하지 않는 쿼리문을 날린다. 나중에 날림(그래서 Lazy)

```java
 @Entity
 public class Member {

     @Id @GeneratedValue
     private Long id;
 
     @Column(name = "USERNAME")
     private String name;
 
     @ManyToOne(fetch = FetchType.LAZY) //****************
     @JoinColumn(name = "TEAM_ID")
     private Team team;
  ..
 }
```

위를 이용하여 만약 `em.find()` 를 한후 `member.getTeam()` 을하면 이때 뭐가 나올까?

-> Team 엔티티의 프록시 객체를 얻을 것이다.

-> 만약 team의 메서드를 호출하는 시점에 직접 DB에 쿼리문을 날리게 될것이다.



+ 만약 지연로딩을 굳이 안할 거라면 ? (딱히 성능상 차이가없어서)
  
  + `fetch = FetchType.Eager` (즉시 로딩)로하거나 관계 매핑 어노테이션의 default 값을 이용하면 된다. 

<br>

### 주의점!

+ **가급적 지연로딩만 사용한다.**

+ 즉시로딩을 하면 예상하지못한 SQL이 발생한다.

+ **가장 치명적인 점은 즉시로딩은 JPQL에서 N+1 문제를 일으킨다는 점이다.**
  
  + 1: 기존 쿼리문 , N: 데이터 개수만큼의 추가 쿼리문
  
  + ex) `em.createQuery("select m from Member m", Member.class)` 1의 JPQL
  
  + 여기에 추가쿼리문 team을 찾는 where절 쿼리문 : 데이터의 갯수만큼 (N 개)
  
  + 물론 해결책은 있다.

+ 관계 매핑 별 default 값 :
  
  + `@ManyToOne`, `@OneToOne` : 기본이 즉시로딩 
    
    + **무조건 LAZY로 설정 해야할 놈들임**
  
  + 나머지 : 기본이 지연로딩.

> 아무튼 지연로딩 아무튼!



<br>

## 7-3 영속성 전이 - CASCADE

+ CASCADE는 쓰이는 상황이 정해져있다. 조심만하면 나름 쓰이는 편이다.

+ 사용이 너무 직관적이라서 한번 보고만 넘어가자.



<u>언제 쓰나요?</u>

+ 특정 엔티티를 영속 상태로 만들때 함계 연관데 엔티티도 함께 영속상태로 만들고 싶을때

+ 연관관계 매핑과는 상관이없음 DB CASCADE느낌이랑 비슷함.

+ 부모만 persist하면 됨.

<u>종류</u>

+ ALL : 모두 적용

+ PERSIST : 영속

+ REMOVE: 삭제.

+ etc..



### 고아 객체

+ 이름이 슬프지만 부모엔티티와 연관관계가 끊어진 자식 엔티티를 자동으로 삭제하는 기능

+ `orphanRemoval = true` 를 사용한다.

+ **참조하는 곳이 하나 일때 사용해야한다.**

+ **즉, 특정 엔티티가 개인 소유할때 사용!**



### 영속성 전이 + 고아객체 = ?

```java
 @OneToMany(mappedBy = "parent",cascade = CascadeType.ALL,orphanRemoval = true)
 private List<Child> childList = new ArrayList<>();
```

+ 두 옵션 모두 활성화하면 마치 부모 엔티티가 자식의 생명주기를 관리하는 것처럼 행동한다.

+ 도메인 주도 설계(DDD)의 Aggreatae Root 개념을 구현할 때 유용하게 사용된다.

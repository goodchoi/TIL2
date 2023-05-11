[인프런 김영한님 강의 - 스프링 DATA JPA](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%EC%8B%A4%EC%A0%84/dashboard)

> 목차
> 
> - [03 스프링 데이터 JPA 구현체 분석](#03---------jpa-------)
>   - [save()](#save--)
>   
>   - [merge()](#merge--)
>   
>   - [Persistable 구현](#persistable---)

# 03 스프링 데이터 JPA 구현체 분석

<img title="" src="https://velog.velcdn.com/images/wonizizi99/post/11f76cbc-1490-488a-b268-e2d8ace2bc17/image.png" alt="" data-align="center" width="553">

이 중에서 구현체인 `SimpleJpaRepository` 를 살펴보자. 그중 `save()` 를 잘봐야한다.

그전에 트랜잭션에 대해 얘기해보자 스프링 데이터 JPA를 사용할때(`JpaRepository`) 트랜잭션을 붙이거나 하지 않았다. 그이유는,,

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {
```

+ 이미 구현체에 트랜잭션을 붙여주기 때문이다. 클래스 레벨에서 `readOnly = true` 를 한다는 뜻은 저장,수정,삭제 에서는 `@Transactional` (default -> false)를 붙이겠다는 뜻이다.
+ 만약 서비스에서 트랜잭셔을 먼저 수행할 경우, 트랜잭션 전파로 내부 트랜잭션이 될것이다.

#### save()

```java
    @Transactional
    @Override
    public <S extends T> S save(S entity) {

        Assert.notNull(entity, "Entity must not be null.");

        if (entityInformation.isNew(entity)) {
            em.persist(entity);
            return entity;
        } else {
            return em.merge(entity);
        }
    }
```

+ 기본편에서 순수 JPA 를 사용하여 save를 구현할때도 위와 비슷했다. 여기서 중요한점은 `entityInformation.isNew(entity)` 이다. 즉, 새로운 엔티티이면 `persist` 하고, 아니면 `merge` 하겠다는 것인데,  뭘 기준으로 새로운 엔티티 인지 판단하냐는 것이다.



+ `isNew()` 기본 전략 : 식별자를 기준으로한다.
  
  + 식별자가 객체일때 : null이면 새로운 엔티티
  
  + 식별자가 원시타입일때 : 0이면 새로운 엔티티
  
  + `Persistabel` 인터페이스를 구현하여 판단 로직으로 새로운 엔티티 판단.



> 기본키 생성전략을 `@GeneratedValue` 라면 당연히 save() 호출 시점에 식별자가 없으므로 새로운 엔티티임은 자명하다. 즉 위를 고려 하지 않아도 된다.
> 
> 근데 만약 `@Id` 만 사용하고 ,식별자값을 직접 할당하는 구조라면? (기본편에서 식별자 테이블 ) 위를 고려하지 않으면 `persist()` 가 아닌 `merge()` 를 호출한다. 



#### merge()

+ 기본편에서도 배웠지만 merge는 그냥 일반 수정이랑은 약간 다르다. merge는 어이없게 DB 에 select문을 호출해서 값을 확인하는 작업이 추가된다. -> 매우 비효율적

+ 모든 값을 바꿔치기 한다.



#### Persistable 구현

```java
@EntityListeners(AuditingEntityListener.class)
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item implements Persistable {

    @Id
    private String id;

    @CreatedDate
    private LocalDateTime createdDate;

    public Item(String id) {
        this.id = id;
    }


    @Override
    public boolean isNew() {
        return createdDate == null;
    } //!
}
```

`isNew` 메서드를 구현해주면 되는데 이때 가장 효율적인 방법은 생성 시간이 null 인지 여부를 알려주면되는것이다.

-> 생성시간은 `persist()` 할때 호출되므로 `save()` 를 호출하는 시점에는 당연히 Null 이기 때문이다.

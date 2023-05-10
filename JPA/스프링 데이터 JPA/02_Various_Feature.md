[인프런 김영한님 강의 - 스프링 DATA JPA](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%EC%8B%A4%EC%A0%84/dashboard)

# 02  확장 기능

## 02-1 사용자 정의 리포지토리 구현

스프링 데이터 JPA 리포지토리 인터페이스를 사용하면서 내가 직접 메서드를 구현하고 싶다면? 

잘생각해보자, <u>인터페이스는 구현하려면 인터페이스에 정의된 메서드를 모두 구현해야한다. </u> 그게아니라 한두개 정도만 모종의 이유로 직접 구현하고 싶다면?

모종의 이유:

+ 순수 JPA사용 ,

+ 스프링 JDBC TEMPLATE 사용, 마이바티스  ,** Quertdsl** 사용 등

#### 사용자 정의 인터페이스

```java
public interface MemberRepositoryCustom {
    List<Member> findMemberCustom();
}
```

<br>

#### 사용자 정의 인터페이스 구현체

```java
package study.datajpa.repository;

import lombok.RequiredArgsConstructor;
import study.datajpa.entity.Member;

import javax.persistence.EntityManager;
import java.util.List;

@RequiredArgsConstructor
public class MemberRepositoryImpl implements MemberRepositoryCustom{

    //굳이 커스텀에 집착하지말자 . 어차피 커스텀으로 만들어도 다 MemberRepository에 속하게된다.
    //차라리 핵심로직과 화면에 맞춘 쿼리는 분리를 하자.
    //심지어 인터페이스 없이 클래스로 만들어서 빈으로 등록하면된다.

    private final EntityManager em;

    @Override
    public List<Member> findMemberCustom() {
        return em.createQuery("select m from Member m",Member.class)
                .getResultList();
    }
}
```

<br>

#### 사용자 정의 인터페이스 상속

```java
public interface MemberRepository extends JpaRepository<Member, Long> , MemberRepositoryCustom 

  ////////////////////////////////////////////////////////

}
```

+ 지금 이 행위는 리포지토리를 새로 빈으로 등록하는 행위가 아니다. 스프링 데이터 JPA인터페이스를 사용하되, 거기에 내가 원하는 구현 메서드를 추가 하고 싶은 상황이다.  

+ 인터페이스에 일반 Class 를 상속 할 수 있겠는가? 그게 안되기 때문에 새로 인터페이스를 만들고 이것을 상속 시키는 것이다. 이때, 우리가 구현한 클래스로 메서드를 생성하는 것은 오로지 스프링 데이터 JPA 의 영향이다.

##### 주의 : 구현하는 클래스의 이름은 규칙이있다. `MemberRepositoryImpl`  와 같이 원래 리포지토리 이름 + `Impl` 이다. 이렇게 해야만 인식한다. 추가) 추가한 인터페이스의 이름에 `Impl` 을 붙여도 인식한다. ex)`MemberRepositoryCustomImpl`

> 참고 : 커스텀하는 행위에 대해서 활용편에 설명했듯이 딱히 좋은 행위가 아니다. 앞에서 배웠던 내용에 의하면 ,핵심 비즈니스 로직과 화면에 핏하게 들어맞는 쿼리 로직을 분리하는게 좋다고 했다. 정확히는 리포지토리 자체를 분리시켜서 둘다 빈으로 등록하는 것이다. 

<br>

## 02-2 Auditing

강의에서 언급하기를, 실무에서 거의 모든테이블에  다음과 같은 컬럼이 들어있으면 유지보수에 도움을 준다고 했다.

+ 등록일

+ 수정일

+ 등록자

+ 수정자

#### 순수 JPA 사용

```java
@MappedSuperclass
@Getter
public class JpaBaseEntity {

    @Column(updatable = false)//update못하게 막는 기능
    private LocalDateTime createdDate;
    private LocalDateTime updatedDate;

    @PrePersist
    public void prePersist() {
        LocalDateTime now = LocalDateTime.now();
        createdDate = now;
        updatedDate = now;
    }

    @PreUpdate
    public void preUpdate() {
        updatedDate = LocalDateTime.now();
    }

}
```

#### 엔티티

```java
@Entity
public class Member extends BaseEntity{ }
```

---

#### 스프링 데이터 JPA 사용(부트 설정 클래스)

```java
@EnableJpaAuditing //달아 줘야
@SpringBootApplication
public class DataJpaApplication {

    public static void main(String[] args) {
        SpringApplication.run(DataJpaApplication.class, args);
    }

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.of(UUID.randomUUID().toString());
        //AuditorAware 내부의 getCurrentAuditor()를 람다로 구현
        //스프링 시큐리티 연동해보는 포인트
    }
}
```

#### 스프링 데이터 Auditing 적용

```java
@EntityListeners(AuditingEntityListener.class) //등록자, 수정자 때문에 필
@MappedSuperclass
@Getter
public class BaseTimeEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;

    @LastModifiedDate
    private LocalDateTime lastModifiedDate;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String lastModifiedBy;

}
```

+ 스프링 데이터 JPA는 기존에 JPA로 통상적으로 작성되는 코드를 이미 라이브러리에 저장해두어서 사용자들의 편의성을 증대하는것이 존재의의 같다.
+ 지금도 어노테이션을 달아줌으로써 딱히 메서드를 지정해주지 않아도 된다.
+ 물론, 작성자, 수정자의 경우 보통 세션을 이용하여 값을 가져오거나 <mark>스프링 시큐리티와 연동하</mark>여 값을 가져와야하기 때문에 직접 구현해주는데 이때도, 공통적으로 처리하기위해 스프링 설정 파일에 그것을 명시한다.
  + 나중에 스프링 시큐리티를 배워서 한번 써먹어보자.

> 참고로 대부분의 엔티티는 등록시간 수정시간이 필요하지만, 등록자 ,수정자는 필요없을 수도 있다. 이럴 땐, 둘을 따로 분리해서 등록자 수정자 쪽에서 상속받으면된다.

ex)

```java
public class BaseTimeEntity {
     @CreatedDate
     @Column(updatable = false)
     private LocalDateTime createdDate;
     
     @LastModifiedDate
     private LocalDateTime lastModifiedDate;
}

public class BaseEntity extends BaseTimeEntity {
     @CreatedBy
     @Column(updatable = false)
     private String createdBy;
     @LastModifiedBy
     private String lastModifiedBy;
}
```

<br>





## 02-3 Web-확장 도메인 클래스 컨버터

컨트롤러에서 파라미터로 Id를 넘겨받을때(Rest 스타일) 간편하게 엔티티객체를 찾아서 바인딩해주는 기능이있다.

```java
@GetMapping("/members2/{id}")
    public String findMember2(@PathVariable("id") Member member) {
        return member.getUsername();
    } //권장하진 않음.
```

+ 알아서 쿼리문 나가고 엔티티객체 바인딩까지 해준다.

+ 권장하지 않고 ,만약 사용한다면 단순조회 용도만사용해야한다. 트랜잭션 범위에서 실행되지 않기 때문이다.

<br>

## 02-4 Web-확장 페이징과 정렬

스프링데이터가 제공하는 페이징과 정렬기능을 MVC에서 편리하게 사용할 수 있다.

```java
@GetMapping("/members")
    public Page<MemberDto> list(@PageableDefault(size = 5,sort = "username",direction = Sort.Direction.DESC) Pageable pageable) {
        Page<Member> page = memberRepository.findAll(pageable);
        return page.map(member -> new MemberDto(member.getId(), member.getUsername(), null));
    }
```

+ 기본적으로 `Pageable` 을 파라미터로 받을 수 있는데 이부분은 아마 스프링 MVC에서 배운 `ArgumentResolver` 가 제공하는 것아닐까 싶다. 실제로는 `Pageable`은 인터페이스이므로 객체는 `org.springframework.data.domain.PageRequest` 를 생성해서 넣어주는 것 같다.

+ `@PageableDefault` 로 기본 페이지 설정을 할 수 있다. 없어도 됨. 없으면 size 디폴트는 20이다.

+ **주의할점은 스프링에서 Page 인덱스는 0부터 시작한다는 점이다.**

+ 위의 코드에서는 Dto로 변환하는 예시이다.

#### 요청 파라미터 예시

ex) `/members?page=0&size=3&sort=id,desc&sort=username,desc` 

+ 0번째 페이지, 3건조회, id,username 을 기준으로 내림차순.



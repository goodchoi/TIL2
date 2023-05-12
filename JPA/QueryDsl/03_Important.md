[실전! Querydsl 대시보드 - 인프런 | 강의 (inflearn.com)](https://www.inflearn.com/course/querydsl-%EC%8B%A4%EC%A0%84/dashboard)



# 03 중급 문법



## 03-1 프로젝션과 결과 반환 - 기본

프로젝션이란 select 대상 을 직접 지정해서 가져올떄 쓰는 표현이다.

#### 프로젝션 대상이 하나일때

```java
@Test
void simpleProjection() {

    List<String> result = queryFactory
            .select(member.username)
            .from(member)
            .fetch();

    System.out.println("result = " + result);
}
```

+ 프로젝션 대상이 하나이면 타입을 명확하게 지정 해 줄 수 있다. 

+ 만약 둘 이상이면 튜플이나 DTO를 사용하여 조회한다.



#### 튜플로 조회

```java
@Test
void tupleProjection() {

    List<Tuple> result = queryFactory
            .select(member.username, member.age)
            .from(member)
            .fetch();

    for (Tuple tuple : result) {
        Integer age = tuple.get(member.age);
        String username = tuple.get(member.username);

        System.out.println("username = " + username);
        System.out.println("age = " + age);

    }
//튜플 같은 특정 기술에 종속적인 객체들은 리포지토리에서만 사용하는것은 결코 좋지않다.
//dto 객체로 변환해서 던지던지(?) 해야한다.
```

+ 파이썬의 튜플과 비슷한 개념을 Querydsl이 도입했다. 즉, 타입이 다른 쿼리결과를 가져온다.

+ 하지만 튜플은, `Querydsl` 기술이다. 만약 서비스 계층, 컨트롤러에서 튜플을 다루게 된다면 JPA  및 Querydsl에 종속적이게 되버린다. 따라서 튜플은 리포지토리에서만 사용하자.

<br>

## 03-2 프로젝션과 결과 반화 - DTO 조회

#### MeberDto

```java
@Data
@NoArgsConstructor
public class MemberDto {

    private String username;
    private int age;

    @QueryProjection //dto도 Q 타입으로 생성
    public MemberDto(String username, int age) {
        this.username = username;
        this.age = age;
    }
}
```



##### 결과를 Dto로 반환하는 방법

1. 프로퍼티 접근

2. 필드 직접 접근

3. 생성자 사용(+ @QuertPrjoection 사용)



#### 프로퍼티 접근 - Setter로 접근함

```java
@Test
void findDtoBySetter() {
    List<MemberDto> result = queryFactory
            .select(Projections.bean(MemberDto.class,
                    member.username,
                    member.age))
            .from(member)
            .fetch();

    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
```



#### 필드 직접 접근

```java
@Test
void findDtoByField() {
    List<MemberDto> result = queryFactory
            .select(Projections.fields(MemberDto.class,
                    member.username,
                    member.age))
            .from(member)
            .fetch();

    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }

    // 기본생성자 필요
}
```

+ 이 방법은 getter,setter 로 하는게 아니다. 리플렉션 등의 기술로 직접 필드에 초기화 시켜버린다.



> 참고 
> 
> 1번 방식 이나 2번 방식의 경우 필드 이름과 , 조회하는 엔티티의 필드이름이 다를 경우, as로 별칭을 같게 해주면 된다.(생성자의 경우 상관 x)





#### 생성자 사용

```java
@Test
void findDtoByConsturctor() {
    List<MemberDto> result = queryFactory
            .select(Projections.constructor(MemberDto.class,
                    member.username,
                    member.age))
            .from(member)
            .fetch();

    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
```



#### 생성자 + @QueryProjection

위의 dto에서 생성자에 `@QueryProjection` 이 붙은 것을 확인할 수 있다. 이렇게 하면 엔티티가 아님에도 Q타입이 생성된다. 

```java
@Test
void findDtoByQueryProjection() {
    List<MemberDto> result = queryFactory
            .select(new QMemberDto(member.username, member.age)) //컴파일 시점에 실수 방지 가능 위의 생성자 방식은 컴파일 단계에 확인불가.
            .from(member)
            .fetch();

    for (MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
```

+ 이방식의 단점 :
  
  + 큐파일을 생성해야한다.
  
  + `dto`가 `Querydsl` 에 의존하게 된다. (내부에 `@QueryProjection` 을 가지게되므로)
    
    + dto가 더럽혀지는 꼴 못보는 개발자들이 있다.

+ 그럼에도 압도적으로 편하고 안전하다. 컴파일 시점에 타입체크를 할 수 있다.

<br>

## 03-3 동적쿼리 - BooleanBuilder 사용

동적 쿼리를 해결하는 데에는 두가지 방법이있다.

+ BooleanBuilder

+ Where 다중 파라미터 사용 - (선호)



```java
@Test
void dynamicQuery_BooleanBuilder() {
    String usernameParam = "member1";
    Integer ageParam = 10;

    List<Member> result = searchMember1(null, ageParam);

    assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember1(String usernameCond, Integer ageCond) {

    BooleanBuilder builder = new BooleanBuilder();
    if (usernameCond != null) {
        builder.and(member.username.eq(usernameCond));
    }

    if (ageCond != null) {
        builder.and(member.age.eq(ageCond));
    }

    return queryFactory
            .selectFrom(member)
            .where(builder)
            .fetch();
}
```

Quertdsl 자체가 JPA를 사용하여 동적쿼리를 해결하는 방법중에는 끝판왕이다. 하지만 BooleanBuilder는 직관적으로 코드를 이해하기 조금 쉽지 않은 감이 있다.

<br>

## 03-4 동적쿼리 - Where 다중 파라미터 사용

```java
@Test
void dynamicQuery_WhereParam() {
    String usernameParam = "member2";
    Integer ageParam = null;

    List<Member> result = searchMember2(usernameParam, ageParam);

    assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember2(String usernameCond, Integer ageCond) {

    return queryFactory
            .selectFrom(member)
            //.where(usernameEq(usernameCond), ageEq(ageCond))
            .where(allEq(usernameCond,ageCond))
            .fetch();
}

//    private Predicate usernameEq(String usernameCond) {
//        return usernameCond != null ? member.username.eq(usernameCond) : null;
//    } BooleanExpression 을 쓰는게 낫다.

private BooleanExpression usernameEq(String usernameCond) {
    return usernameCond != null ? member.username.eq(usernameCond) : null;
}

private BooleanExpression ageEq(Integer ageCond) {
    return ageCond != null ? member.age.eq(ageCond) : null;
}

private BooleanExpression allEq(String usernameCond , Integer ageCond) {
    return usernameEq(usernameCond).and(ageEq(ageCond));
    //조립을 이 방식을 이용하면 조립을 할 수 있다. 재사용을 할 수 있다.
    //가독성이 높아짐.
}
```



+ 참고로 메서드 뽑을때 자동으로 리턴타입을 `Predicate` 를 반환하게 하는데 , 이거 말고 `BooleanExpression` 으로 반환타입을 정해주면 좋다. 메서드의 재조합이 가능하기 떄문이다.

+ 쿼리 자체의 가독성이 올라간다.

+ 메서드를 뽑아쓰기 떄문에 코드 재사용성이 생긴다.

+ 심지어 메서드끼리 조합도 가능하다.

+ 그리고 이렇게하면 재밌다..

<br>



## 03-5 SQL function 호출

Dialect에 등록된 함수만 호출 가능. 사용자 정의함수 등록하는 법도 배웠다는걸 잊지말자



```java
@Test
void sqlFunction() {
    List<String> result = queryFactory
            .select(
                    Expressions.stringTemplate("function('replace', {0}, {1}, {2})"
                            , member.username, "member", "M")) //dialect에 등록된 function만
                            // member.username 을 대상으로 , 'member'를 'M'으로 바꿔라
            .from(member)
            .fetch();

    for (String s : result) {
        System.out.println("s = " + s);
    }
}
```






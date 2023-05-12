[실전! Querydsl 대시보드 - 인프런 | 강의 (inflearn.com)](https://www.inflearn.com/course/querydsl-%EC%8B%A4%EC%A0%84/dashboard)

# 04 실무 활용 - 순수 JPA와 Query dsl

## 04-1 순수 JPA 리포지토리와 Querydsl

#### 순수 JPA리포지토리

```java
@Repository
@RequiredArgsConstructor
public class MemberJpaRepository {


    private final EntityManager em;

    public void save(Member member) {
        em.persist(member);
    }

    public Optional<Member> findById(Long id) {
        Member member = em.find(Member.class, id);
        return Optional.ofNullable(member);
    }

    public List<Member> findAll() {
        return em.createQuery("select m from Member m",Member.class)
                .getResultList();
    }

    public List<Member> findByUsername(String username) {
        return em.createQuery("select m from Member m " +
                "where m.username = :username",Member.class)
                .setParameter("username",username)
                .getResultList();
    }
```

#### Querydsl 사용

```java

@Repository
@RequiredArgsConstructor
public class MemberJpaRepository {


    private final EntityManager em;
    private final JPAQueryFactory queryFactory;



    public List<Member> findAll_QueryDsl(){
        return queryFactory
                .selectFrom(member)
                .fetch();
    }

    public List<Member> findByUsername_Querydsl(String username) {
        return queryFactory
                .selectFrom(member)
                .where(member.username.eq(username))
                .fetch();
    }

    public List<MemberTeamDto> seachBuilder(MemberSearchCondition condition) {

        BooleanBuilder builder = new BooleanBuilder();

        if(hasText(condition.getUsername())) {
            builder.and(member.username.eq(condition.getUsername()));
        }

        if (hasText(condition.getTeamname())) {
            builder.and(team.name.eq(condition.getTeamname()));
        }

        if (condition.getAgeGoe() != null) {
            builder.and(member.age.goe(condition.getAgeGoe()));
        }

        if (condition.getAgeLoe() != null) {
            builder.and(member.age.loe(condition.getAgeLoe()));
        }

        return queryFactory
                .select(new QMemberTeamDto(
                        member.id.as("memberId"),
                        member.username,
                        member.age,
                        team.id.as("teamId"),
                        team.name.as("teamName")
                ))
                .from(member)
                .where(builder)
                .leftJoin(member.team, team)
                .fetch();
    }

    public List<MemberTeamDto> search(MemberSearchCondition condition) {
        return queryFactory
                .select(new QMemberTeamDto(
                        member.id.as("memberId"),
                        member.username,
                        member.age,
                        team.id.as("teamId"),
                        team.name.as("teamName")
                ))
                .from(member)
                .where(
                        usernameEq(condition.getUsername()),
                        teamnameEq(condition.getTeamname()),
                        ageGoe(condition.getAgeGoe()),
                        ageLoe(condition.getAgeLoe())
                )
                .leftJoin(member.team, team)
                .fetch();
    }

    private BooleanExpression usernameEq(String username) {
        return hasText(username) ? member.username.eq(username) : null;

    }

    private BooleanExpression teamnameEq(String teamname) {
        return hasText(teamname) ? team.name.eq(teamname) : null;
    }

    private BooleanExpression ageGoe(Integer ageGoe) {
        return ageGoe != null ? member.age.goe(ageGoe) : null;
    }

    private BooleanExpression ageLoe(Integer ageLoe) {
        return ageLoe != null ? member.age.loe(ageLoe) : null;
    }

    private BooleanExpression ageBetween(int ageLoe, int ageGoe) {
        return ageGoe(ageGoe).and(ageLoe(ageLoe));
    }

}
```

+ 참고로 스프링 Application 파일에서 `JPAQueryFactory` 를 빈으로 등록했다.

Querydsl은....꿀잼이다.



딱히 설명할 게 없군..



아 참고로, 동적쿼리를 만들때, 가끔 데이터가 감당안되게 많은 경우 파라미터가 아무것도 없으면 데이터가 전체 조회 되는데, 이는 리소스를  너무 소모하므로 limit를 거는것을 고려하기도 한다.







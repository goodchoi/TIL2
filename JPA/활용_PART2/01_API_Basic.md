[인프런 - 김영한님 강의 스프링 부트와 JPA 활용 part2 ](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94)

# 01 API 개발 기본

## 

## 회원 등록 API

### version1

* RequestBody에 직접 매핑하기

```java
 @PostMapping("/api/v1/members")
    public CreateMemberResponse saveMember(@RequestBody @Valid Member member) {
        Long id = memberSerivce.join(member);
        return new CreateMemberResponse(id);
    }
```

+ 문제점
  
  + validation Rule을 엔티티에 걸어야하는 상황이 생겨버린다. 
  
  + 엔티티를 수정하면 API스펙이 변경되는 대참사가 일어난다.
  
  + API 스펙과 정확하게 일치하지않는다.

+ 결론 
  
  + DTO를 사용하자!

### version2

+ DTO를 RequestBody에 매핑하기

```java
    @PostMapping("/api/v2/members")
    public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request) {

        Member member = new Member();
        member.setName(request.getName());

        Long join = memberSerivce.join(member);
        return new CreateMemberResponse(join); //별도의 Request객체와 resoponse 객체를 이용하고 있음.
    }


    @Data
    static class CreateMemberRequest {
        @NotEmpty
        private String name;
    }

    @Data
    static class CreateMemberResponse {
        private Long id;

        public CreateMemberResponse(Long id) {
            this.id = id;
        }
    }
```

+ 엔티티와 API스펙을 명확하게 분리할 수있다.

+ 엔티티가 변해도 API스펙이 변하지않음

+ 엔티티와 프레젠테이션 계층을 위한 로직을 분리할 수 있다.

> 실무에서는 엔티티를 API스펙에 노출하면 안된다!!!

<br>

## 회원 수정 API

```java
    @PutMapping("/api/v2/members/{id}")
    public UpdateMemberResponse updateMemberV2(
            @PathVariable("id") Long id,
            @RequestBody @Valid UpdateMemberRequest request) {

        memberSerivce.update(id, request.getName()); //변경감지 정의
        Member findmember = memberSerivce.findOne(id); //특별히 트래픽이 많은 API 가 아니면 크게 이슈는 없다.
        return new UpdateMemberResponse(findmember.getId(), findmember.getName());
    }


    @Data
    static class UpdateMemberRequest {
        private String name;
    }

    @Data
    @AllArgsConstructor
    static class UpdateMemberResponse {
        private Long id;
        private String name;
    }
```

+ 웹 어플리케이션과 다르게 API 통신은 수정을 `PUT` `PATCH` `POST`를 이용할 수 있다. 

+ 변경 감지를 사용해서 데이터를 수정한다.

> 전체 업데이트는 PUT 메서드를 사용하고, 부분 업데이트시 PATCH 혹은 POST를 사용하는것이 REST 스타일에 맞다.



+ MemberService에 update메서드를 새로 만들었음 이때, 그냥 반환을 Member객체로 하면 되지않나 싶은데 유지보수적 관점에서 데이터가 변경되는 메서드와 조회메서드로 크게분리했을때
  조회 메서드는 데이터가 변경되지않게 설계하는것이좋고,
  update같은 메서드는 아예 데이터를 반환하지않는게 좋다. 그래서 위의 코드에서 id값으로 새로 조회를 하고있다.

<br>

## 회원 조회 API

### version1

```java
    @GetMapping("/api/v1/members")
    public List<Member> membersV1() {
        return memberSerivce.findmembers();
    }
```

+ 문제점
  
  + 엔티티의 모든값이 노출된다.
  
  + 응답 스펙을 맞추기위해 (@JsonIgnore) 같은 로직이 추가된다.
  
  + 엔티티가 변경되면 API스펙이 변한다. 

## version2

```java
   @GetMapping("/api/v2/members")
    public Result membersV2() {
        List<Member> findmembers = memberSerivce.findmembers();
        List<MemberDTO> collect = findmembers.stream()
                .map(m -> new MemberDTO(m.getName()))
                .collect(Collectors.toList());

        return new Result(collect.size(),collect);
    }

    @Data
    @AllArgsConstructor
    static class Result<T> {
        private int count;
        private T data;
    }

    @Data
    @AllArgsConstructor
    static class MemberDTO {
        private String name;
    }
```

+ 엔티티를 전부 DTO로 변환후 반환한다.

+ 엔티티가 변해도 API 스펙이 변경되지 않음.

+ `Result` 클래스로 컬렉션을 한번 감싸는 것은 API스펙에 필드가 추가될수 있게한다.

+ 이렇게안하면 무조건 List만 반환할것임.

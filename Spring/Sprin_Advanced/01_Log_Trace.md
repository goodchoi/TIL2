[인프런 김영한님 강의 - 스프링_고급](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8/dashboard)



# 01 로그 추적기

> 스프링 고급편 강의 섹션1과 섹션2를 요약



## 01-1 로그 추적기

큰 규모의 프로젝트는 모니터링과 운영이 중요하다. 특히 병목 현상이나 예외가 어디서 발생하는 지 쉽게확인하는 방법은 로그를 미리 남겨놓는 것이다. 

#### 로그 추적기 예시

```log
정상 요청
  [796bccd9] OrderController.request()
  [796bccd9] |-->OrderService.orderItem()
  [796bccd9] |   |-->OrderRepository.save()
  [796bccd9] |   |<--OrderRepository.save() time=1004ms
  [796bccd9] |<--OrderService.orderItem() time=1014ms
  [796bccd9] OrderController.request() time=1016ms

예외 발생
  [b7119f27] OrderController.request()
  [b7119f27] |-->OrderService.orderItem()
  [b7119f27] | |-->OrderRepository.save()
  [b7119f27] | |<X-OrderRepository.save() time=0ms
  ex=java.lang.IllegalStateException: 예외 발생!
  [b7119f27] |<X-OrderService.orderItem() time=10ms
  ex=java.lang.IllegalStateException: 예외 발생!
  [b7119f27] OrderController.request() time=11ms
  ex=java.lang.IllegalStateException: 예외 발생!
```

이런 식으로 메서드의 depth와 실행시간, 트랜잭션 ID, 예외 내용 등을 한번에 알아보게끔 로그 추적기를 만들려고한다.





#### HelloTraceV1

```java
@Slf4j
@Component
public class HelloTraceV1 {

    private static final String START_PREFIX = "-->";
    private static final String COMPLETE_PREFIX = "<--";
    private static final String EX_PREFIX = "<X-";

    public TraceStatus begin(String message) {
        TraceId traceId = new TraceId();
        Long startTimeMs = System.currentTimeMillis();
        log.info("[{}] {}{}", traceId.getId(), addSpace(START_PREFIX, traceId.getLevel()), message);
        return new TraceStatus(traceId, startTimeMs, message);
    }

    public void end(TraceStatus status) {
        complete(status, null);
    }

    public void exception(TraceStatus status, Exception e) {
        complete(status, e);
    }

    private void complete(TraceStatus status, Exception e) {
        Long stopTimeMs = System.currentTimeMillis();
        long resultTimeMs = stopTimeMs - status.getStartTimeMs();
        TraceId traceId = status.getTraceId();
        if (e == null) {
            log.info("[{}] {}{} time={}ms", traceId.getId(), addSpace(COMPLETE_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs);
        } else {
            log.info("[{}] {}{} time={}ms ex={}", traceId.getId(), addSpace(EX_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs, e.toString());
        }
    }

    private static String addSpace(String prefix, int level) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < level; i++) {
            sb.append((i == level - 1) ? "|" + prefix : "|   ");
        }
        return sb.toString();
    }

}
```

#### TraceId

```java
@Getter
public class TraceId {

    private String id;
    private int level;

    public TraceId() {
        this.id = createId();
        this.level = 0 ;
    }

    private TraceId(String id, int level) {
        this.id = id;
        this.level = level;
    }

    private String createId() {
        return UUID.randomUUID().toString().substring(0,8);
        //UUID가 너무길기 떄문에 8 자리만 갖다쓴다.
        //이렇게하면 트랜잭션 ID가 중복될 가능성이 있긴하지만
        //중복될 확률이 극히 낮을 뿐더러 로그기 떄문에 중복되어도 크게 영향이 없다.
    }

    public TraceId createNextId() {
        return new TraceId(id,level+1);
    }

    public TraceId createPreviousId() {
        return new TraceId(id,level-1);
    }

    public boolean isFirstLevel() {
        return level == 0;
    }
```



#### TraceStatus

```java
package hello.advanced.trace;


import lombok.AllArgsConstructor;
import lombok.Getter;

@AllArgsConstructor
@Getter
public class TraceStatus {

    private TraceId traceId;
    private Long startTimeMs;
    private String message;

}

```



#### OrderController

```java

@RestController
@RequiredArgsConstructor
public class OrderControllerV1 {

    private final OrderSerivceV1 orderSerivce;
    private final HelloTraceV1 trace;

    @GetMapping("/v1/request")
    public String request(String itemId) {

        TraceStatus status = null;
        try {

            status = trace.begin("OrderController.request()");
            orderSerivce.orderItem(itemId);
            trace.end(status);
            return "ok";

        } catch (Exception e) {
            trace.exception(status,e);
            throw e; //예외를 꼭 다시 던져줘야한다. catch로 예외를 먹어버리면
                     //정상흐름으로 동작해버린다. 로그가 애플리케이션 흐름에 영향을 주어선안된다.
        }
    }
}
```





+ V1 으로는 아직 로그추적기로서의 기능은 완벽히 하지 못한다.

+ TraceId 가 depth가 내려갈때마다 새로 갱신 되기 때문이다. 

+ 참고로, 예외는 캐치하고 끝내면 정상흐름으로 동작하는데, 지금 이상황에서 try catch문은 로그 때문에 존재한다. 로그는 애플리케이션의 흐름을 방해해서는 안되므로 다시 던져야한다.



<br>

## 로그 추적기 V2 - 파라미터로 동기화

트랜잭션 ID 가 같은 HTTP 요청이면 같은 ID 값을 가져야한다. 이를 위해 파라미터로 계속 넘기는 것을 고려해보자. 물론 해보기도전에 비효율적이라는것을 알고 있다.



#### HelloTraceV2

V1에서 하나만 추가한다.

```java
//V2에서 추가
public TraceStatus beginSync(TraceId beforeTraceId,String message) {

    TraceId nextId = beforeTraceId.createNextId(); //!!
    Long startTimeMs = System.currentTimeMillis();
    log.info("[{}] {}{}", nextId.getId(), addSpace(START_PREFIX, nextId.getLevel()), message);
    return new TraceStatus(nextId, startTimeMs, message);
}
```

+ 기존의 TraceId를 넘겨받아서, depth 만 +1하는 메서드가 추가 되어있다.



#### Controller

```java
    @GetMapping("/v2/request")
    public String request(String itemId) {

        TraceStatus status = null;
        try {

            status = trace.begin("OrderController.request()");
            orderSerivce.orderItem(status.getTraceId(),itemId);
            trace.end(status);
            return "ok";

        } catch (Exception e) {
            trace.exception(status,e);
            throw e;
        }
    }
```

컨트롤러에서는 첫시작이므로 beginSync가 아닌 begin()으로 id값을 생성했고 서비스계층과 리포지토리계층에서는 beginsSync를 사용하면 된다. 또한 컨트롤러 -> 서비스 -> 리포지토리 간의 계층이동간에 `traceId` 를 전달하기위해 메서드에 파라미터를 추가해야한다.



#### 실행 결과

```log
[c80f5dbb] OrderController.request()
[c80f5dbb] |-->OrderService.orderItem()
[c80f5dbb] | |-->OrderRepository.save()
[c80f5dbb] | |<--OrderRepository.save() time=1005ms
[c80f5dbb] |<--OrderService.orderItem() time=1014ms
[c80f5dbb] OrderController.request() time=1017ms
```



## 01-3 남은 문제점

+ 파라미터로 넘기는 방법 말고는 뭐가 있을까? 지금 이방식에서는 만약 서비스를 바로 호출할 경우 TraceId 가 없는데 파라미터로 넘겨야한다.

+ 결국 TraceId를 동기화가 우리의 목표이다. 파라미터로 넘기는 방법말고 TraceId를 동기화 할 방법을 찾아야한다.

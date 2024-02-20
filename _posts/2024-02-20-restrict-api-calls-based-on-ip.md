---
title: IP 기반 API 호출 횟수 제한
categories: [ Spring ]
tags: [ aop ]
---

## 배경

AI 서비스인 Naver Clova Studio를 이용하며 문제가 발생했습니다.
만들고 있던 서비스는 **유저 텍스트 입력 기반 AI 영화추천 앱**입니다. 영화추천을 받을 때마다 토큰을 사용하는데, 비정상적으로 많은 추천을 받아 다량의 비용이 청구된 것을 확인하였습니다.
따라서 유저별로 호출을 제한할 수 있는 장치를 만들어야 했고 해당 경험을 포스팅합니다.
 
## 아이디어

1. 고유 유저별로 호출을 제한
2. Rate Limit 시스템을 구현 (Bucket4j Library)
3. IP 기반으로 호출을 제한

이 중 비로그인 유저도 api를 이용할 수 있어야 했기에 1번은 제외합니다. 또 rate limit 시스템은 충분히 알지 못했기에 2번도 제외합니다. API의 총 호출횟수를 기록하고 싶고, 비로그인 유저도 어느정도 대응을 할 수 있는 3번 아이디어를 선택합니다. 

## AOP를 통한 호출 방지

스프링에서 지원하는 AOP를 통해 기능을 구현해보겠습니다.

#### IpLimit.class

```java
@Getter
@Entity
@Table(name = "ip_limit")
public class IpLimit {

  public static final int DAILY_CALL_LIMIT_COUNT = 3;

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column
  private String ip;

  @Column
  private int totalCallCount;

  @Column
  private int remain;

  public void call() {
    totalCallCount++;
    remain = max(remain - 1, 0);
  }

  public boolean isRunOut() {
    return remain == 0;
  }

  protected IpLimit() {
  }

  public IpLimit(String ip) {
    this.ip = ip;
    this.remain = DAILY_CALL_LIMIT_COUNT;
  }

}
```

유저들의 호출정보를 알 수 있는 객체입니다. IP와 유저의 남은 호출횟수, 총 호출 횟수를 저장합니다.

#### LimitRequest.class

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LimitRequest {
}
```
애너테이션을 기반으로 포인트컷을 지정할 것이기 때문에 그에 맞는 애너테이션을 생성합니다.

#### IpLimitRepository

```java
public interface IpLimitRepository extends JpaRepository<IpLimit, Long> {
    Optional<IpLimit> findByIp(String ip);
}
```

#### LimitRequestAspect.class

```java
@Aspect
@Component
public class LimitRequestAspect {
  private final IpLimitRepository ipLimitRepository;

  public LimitRequestAspect(IpLimitRepository ipLimitRepository) {
    this.ipLimitRepository = ipLimitRepository;
  }

  @Before("@annotation(LimitRequest)")
  public void aop() {
    HttpServletRequest request = getHttpServletRequest();
    String ip = getRequestIpFrom(request);
    IpLimit ipLimit = ipLimitRepository.findByIp(ip).orElse(new IpLimit(ip));
    if (ipLimit.isRunOut()) {
      throw new IllegalArgumentException("호출 횟수를 초과했습니다");
    }
    ipLimit.call();
    ipLimitRepository.save(ipLimit);
  }

  private HttpServletRequest getHttpServletRequest() {
    ServletRequestAttributes servletRequestAttributes = (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
    return servletRequestAttributes.getRequest();
  }

  private String getRequestIpFrom(HttpServletRequest request) {
    String header = request.getHeader("X-Forwarded-For");
    if (header != null) {
      return header.split(",")[0].trim();
    }
    return request.getRemoteAddr();
  }
}
```

1. RequestContextHolder 클래스에 현재 HttpServletRequest를 요청합니다.
2. HttpServletRequest에서 요청자의 IP를 추출합니다.
3. 추출한 IP를 기반으로 IpLimit 객체를 찾고, 없다면 새로 생성합니다.
4. 남은 호출 횟수가 없다면 예외를 발생시킵니다.
5. ipLimit.call()을 호출합니다. 이 때 총 호출 횟수는 증가하고 남은 횟수는 0까지 감소합니다.

## Test
```java
@SpringBootTest
@Import(DemoController.class)
class LimitRequestAspectTest {

    @Autowired
    private DemoController demoController;

    @Test
    void throwExceptionCallApiOverThreeCount() {
        demoController.demo();
        demoController.demo();
        demoController.demo();
        IllegalArgumentException exception = assertThrows(IllegalArgumentException.class, () -> demoController.demo());
        assertThat(exception.getMessage()).isEqualTo("호출 횟수를 초과했습니다");
    }

    @RestController
    static class DemoController {
        @LimitRequest
        @GetMapping
        public void demo() {
        }
    }
}
```
3번을 초과하여 호출하면 예외가 발생하는 테스트를 통과하는 것을 볼 수 있습니다.

## 활용 방안

이렇게 하면 유저는 평생 이 서비스를 3번 밖에 이용하지 못하게 됩니다. 스케줄러를 활용하여 남은 횟수를 매일 초기화 할 수도 있습니다.
```java
// Iplimit class 메서드 추가
// ..
public void reset() {
  remain = DAILY_CALL_LIMIT_COUNT;
}
//..

// IpLimitResetScheduler 추가
@Component
public class IpLimitResetScheduler {
  private static final String EVERYDAY_MIDNIGHT = "0 0 0 * * ?";

  private final IpLimitRepository ipLimitRepository;

  public IpLimitResetScheduler(IpLimitRepository ipLimitRepository) {
    this.ipLimitRepository = ipLimitRepository;
  }

  @Scheduled(cron = EVERYDAY_MIDNIGHT, zone = "GMT+09:00")
  @Transactional
  public void reset() {
    ipLimitRepository.findAll().forEach(IpLimit::reset);
  }
}
```

유저가 매일 요청한 시간을 기점으로 24시간 또는 특정 기간을 설정하고 싶다면 객체를 Redis 저장소에 저장을 한 후 TTL을 이용할 수 있을 것 같습니다.

또한 추후 유저의 등급(프리미엄, 무료)별로 차등제한을 두고 싶다면 세션, 토큰 등 유저 인증정보를 추출해서 작업을 할 수도 있을 것 같습니다.


## 마치며

여기까지 AOP를 활용하여 IP 기반 호출 횟수를 제한 하는 방법을 알아보았습니다. 읽어주셔서 감사합니다. 



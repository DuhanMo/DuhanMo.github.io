---
title: Use Redis to Session Storage
categories: [ Spring ]
tags: [ session, session-storage ]
---

## 들어가며

분산 시스템 환경에서 세션 인증방식의 문제 해결방법 중 하나인 세션 스토리지를 스프링 환경에서 간편하게 구현합니다.  

## 분산 시스템 환경에서 세션 관리 방법

분산 시스템에서 세션을 이용하게 되면 각 서버마다 관리하는 세션아이디가 다르기 때문에 유저는 로그인이 풀리는 경험을 하게 됩니다.

이러한 방식을 막기 위해 대표적으로 세 가지 방법이 존재합니다.

1. Sticky Session
2. Session Clustering
3. Session Storage

이 중 스프링에서 제공하는 기능을 통해 Redis를 Session Storage로 이용해봅니다.


## 의존성 추가

```groovy
implementation 'org.springframework.session:spring-session-data-redis'
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

두 의존성을 모두 추가해주어야 편하게 세션 스토리지를 사용할 수 있습니다.

## .properties 추가

```properties
spring.session.store-type=redis
# 아래 필드 미입력시 기본값 localhost, 6379
spring.data.redis.host=localhost
spring.data.redis.port=6379
```
OR
```yaml
spring:
  session:
    store-type: redis
  # 아래 필드 미입력시 기본값 localhost, 6379
  data:
    redis:
      host: localhost
      port: 6379
```

## 실제 Test

### DemoController

```java
@RestController
public class DemoController {

    @GetMapping("/session")
    public ResponseEntity<Void> setSession(String name, HttpServletRequest request) {
        HttpSession session = request.getSession();
        session.setAttribute("name", name);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/check")
    public ResponseEntity<String> getMemberId(HttpServletRequest request) {
        // getSession(false)
        // session이 없다면 새로 생성 후 반환 하지 않고 null을 반환
        HttpSession session = request.getSession(false);
        Object name = session.getAttribute("name");
        return ResponseEntity.ok("안녕하세요 " + name + "님!");
    }

}
```
### Redis 실행
```shell
# docker redis 컨테이너 실행
docker run -d --name redis -p 6379:6379 redis
```

### 8080, 8081 서버 실행
```shell
# 프로젝트 폴더로 이동
cd ${project-dir}
# gradle로 빌드
./gradlew build

# 2개의 터미널 실행 후 아래 명령어를 통해 2개의 서버를 기동
java -jar myapp.jar --server.port=8080
java -jar myapp.jar --server.port=8081
```
### Redis key 확인
```shell
# docker container terminal 접속
docker exec -it redis /bin/bash
# redis-cli 실행
redis-cli
# 저장된 key 확인
KEYS *
# 출력: (empty array)
```

### 브라우저 재현

브라우저별로 쿠키를 읽고 저장하니 크롬, 사파리 2개를 실행합니다.

- 크롬 -> localhost:8080 접속
- 사파리 -> localhost:8081 접속

#### 크롬 테스트

**크롬으로 먼저 접속해봅니다.**

![img.png](/assets/img/20240225/img.png)

![img_1.png](/assets/img/20240225/img_1.png)

url로 접속 후 쿠키를 확인해보면 서버에서 내려준 SESSION이 저장되어 있는 것을 볼 수 있습니다.

Redis Key를 확인해보면 저장된 키를 확인할 수 있습니다.

![img_2.png](/assets/img/20240225/img_2.png)

#### 사파리 테스트

**사파리로 접속합니다.**

쿠키를 조작해 `localhost:8081/check`로 접속해봅니다.

![img_3.png](/assets/img/20240225/img_3.png)


![img_4.png](/assets/img/20240225/img_4.png)

8080 서버로 접속 후, 로그인 정보를 가진 요청이 8081로 갔을 때 정상적으로 화면이 노출되는 것을 볼 수 있습니다.

## 마치며

Json Web Token을 사용하는 방식과는 다른 Session을 이용한 인증방식을 분산 시스템 환경처럼 재현해보고 결과를 확인해보았습니다.

읽어주셔서 감사합니다.

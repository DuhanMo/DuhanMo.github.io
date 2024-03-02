---
title: "분산 시스템 환경에서 Scheduler 단일 동작시키기"
categories: [ Spring ]
tags: [ shedlock ]
---

## 배경

외부 API를 활용해 데이터를 적재하는 기능을 개발하였습니다. 해당 로직은 스케줄러를 통해 특정한 시각에 동작하도록 개발되었습니다.
하지만 데이터가 중복으로 적재되는 문제가 발생하였습니다.
원인을 파악해 보니 ecs를 통해 다중서버가 동작하고 있어 해당하는 서버에서 스케줄러가 모두 동시에 동작하여 데이터가 중복으로 적재되고 있었습니다.

## 아이디어

1. 데이터베이스에 자체에 Lock을 걸어 여러 스케줄러에서 해당 테이블에 대한 접근 방지
2. 여러 서버에서 스케줄러를 한 개만 동작시키기

해당 테이블에 실시간으로 접근할 수 있는 데이터조차 대기상태로 만들어버릴 수 있기 때문에 1번 방법은 제외하였습니다.

## [ShedLock](https://github.com/lukas-krecan/ShedLock)

ShedLock 라이브러리를 사용하면 스케줄러가 실행하기 전 ShedLock 테이블을 먼저 조회해 Lock을 획득한 스케줄러만 동작하게 합니다.

### 1. 의존성 추가

```groovy
implementation("net.javacrumbs.shedlock:shedlock-spring:4.42.0")
implementation("net.javacrumbs.shedlock:shedlock-provider-jdbc-template:4.42.0")
```

### 2. Configuration 클래스에 애너테이션 추가

```java

@SpringBootApplication
@EnableScheduling // 스케줄러 사용을 위한 애너테이션
@EnableSchedulerLock(defaultLockAtMostFor = "60m") // ShedLock 사용을 위한 애너테이션
public class Application {

  public static void main(String[] args) {
    SpringApplication.run(InfluencercardApplication.class, args);
  }

}
```

`@EnableSchedulerLock`을 추가합니다. `defaultLockAtMostFor` 옵션은 서버가 내려가도 해당 락이 가지는 기본 최대 시간을 정하는 옵션입니다. (필수)
스케줄러별로 `lockAtMostFor` 옵션을 설정할 수 있습니다.

### 3. 실제 실행될 메서드에 애너테이션 추가

```java

@Scheduled(cron = "0 0 04 * * MON", zone = "GMT+09:00")
@SchedulerLock(name = "scheduleTask", lockAtMostFor = "60m", lockAtLeastFor = "30m")
public void scheduleTask() {
  fooService.something();
}
```

`@SchedulerLock`을 추가합니다. `lockAtMostFor` 옵션을 설정하지 않으면 `@EnableSchedulerLock`의 옵션으로 주었던 디폴트가 적용됩니다.
`lockAtLeastFor`는 매우 짧은 주기로 진행되는 스케줄러가 락을 갖고 있을 최소 시간을 정해주는 옵션입니다.

### 4. Provider 세팅 (DB & Bean)

#### create table

```sql
# MySQL, MariaDB

CREATE TABLE shedlock
(
  name       VARCHAR(64)  NOT NULL,
  lock_until TIMESTAMP(3) NOT NULL,
  locked_at  TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  locked_by  VARCHAR(255) NOT NULL,
  PRIMARY KEY (name)
);
```

MySQL, MariaDB 기준 위의 테이블이 필요합니다.

#### spring bean

```java

@Configuration
public class ShedLockConfig {

  @Bean
  public JdbcTemplateLockProvider lockProvider(DataSource datasource) {
    return JdbcTemplateLockProvider(
      JdbcTemplateLockProvider.Configuration.builder()
        .withJdbcTemplate(JdbcTemplate(datasource))
        .usingDbTime()
        .build()
    );
  }

}
```

위와 같이 bean을 설정합니다.

위 세팅 진행 후 스케줄러를 동작시켜 보면 아래와 같이 `shedlock` 테이블에 데이터가 쌓입니다.

| name         | lock_until                    | locked_at                     | locked_by |
|--------------|-------------------------------|-------------------------------|-----------|
| scheduleTask | 2022-12-18 19:30:00.104 (UTC) | 2022-12-18 19:00:00.104 (UTC) | unknown   |

최초 스케줄러 실행 시 해당 row가 insert 됩니다. 이후 스케줄러가 진행될 때마다 lock_until, locked_at 컬럼이 업데이트됩니다.

> Warning Do not manually delete lock row from the DB table. ShedLock has an in-memory cache of existing lock rows so
> the row will NOT be automatically recreated until application restart. If you need to, you can edit the row/document,
> risking only that multiple locks will be held.

응용 프로그램을 다시 시작할 때까지 행이 자동으로 생성되지 않기 때문에 수동으로 삭제하지 말라고 주의를 주고 있습니다.

## 마치며

분산 시스템 환경에서 Scheduler 단일 동작시키는 방법을 알아보았습니다. 읽어주셔서 감사합니다.

## reference
> [https://github.com/lukas-krecan/ShedLock](https://github.com/lukas-krecan/ShedLock)

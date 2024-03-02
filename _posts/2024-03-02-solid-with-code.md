---
title: "객체지향 5원칙 SOLID 코드로 이해하기"
categories: [ Programming ]
tags: [ solid ]
---

## 들어가며

객체지향 5원칙을 실제 예시를 통해 알아봅니다.

- S: 단일 책임 원칙 (Single Responsibility Principle, SRP)
- O: 개방-폐쇄 원칙 (Open-Closed Principle, OCP)
- L: 리스코프 치환 원칙 (Liskov Substitution Principle, LSP)
- I: 인터페이스 분리 원칙 (Interface Segregation Principle, ISP)
- D: 의존관계 역전 원칙 (Dependency Inversion Principle, DIP)

## 단일 책임 원칙 (Single Responsibility Principle, SRP)

### 요점

- 하나의 객체가 하나의 책임만 가져야 한다.
- 클래스는 단 한 가지 목표한 가지고 작성해야 한다.
- 애플리케이션 모듈 전반에서 높은 유지보수성과 가시성 제어 기능을 유지하는 원칙이다.

### 예시

**사각형면적계산기**를 예로 들겠습니다.

#### SRP를 따르지 않은 경우

```java
public class RectangleCalculator {
  private static final double IN애_TERM = 0.0254d;
  private final int width;
  private final int height;

  public RectangleCalculator(int width, int height) {
    this.width = width;
    this.height = height;
  }

  public int area() {
    return width * height;
  }

  public double metersToInches(int area) {
    return area / INCH_TERM;
  }
}
```

면적 계산기에서 단위변환 일까지 하고있습니다. 면적계산이라는 클래스의 목적과 부합하지 않아 유지보수성이 떨어지므로 분리해봅니다.

#### SRP를 따른 경우

```java
public class RectangleCalculator {
  private final int width;
  private final int height;

  public RectangleCalculator(int width, int height) {
    this.width = width;
    this.height = height;
  }

  public int area() {
    return width * height;
  }
}
```

```java
public class AreaConverter {
  private static final double INCH_TERM = 0.0254d;
  private static final double FEET_TERM = 0.3048d;

  public double metersToInches(int area) {
    return area / INCH_TERM;
  }

  public double metersToFeet(int area) {
    return area / FEET_TERM;
  }
}
```

각 클래스가 맡은 하나의 일만 하기 때문에 단일 책임 원칙을 따릅니다.

SRP를 지켜 코드를 작성하면, 각 클래스가 어떤 일을 할지 예측가능합니다. 예측가능한 코드를 짜는 것은 동료 개발자, 미래의 본인의 인지적 부하를 줄이고 유지보수에 용이하기 때문에 중요하다고 생각합니다.

## 개방-폐쇄 원칙 (Open-Closed Principle, OCP)

### 요점

- 소프트웨어 컴포넌트는 확장에 관해 열려있고, 수정에 관해서는 닫혀 있어야 한다.
- 다른 개발자가 클래스를 확장하기만 해도 원하는 작업을 할 수 있도록 해야한다.

### 예시

**도형 인터페이스, 도형의 면적계산기**를 예로 들겠습니다.

#### OCP를 따르지 않은 경우

```java
public interface Shape {
}

public class Rectangle implements Shape {
  private final int width;
  private final int height;
  // constructor, getter, setter ..
}
```

```java
public class ShapeAreaCalculator {
  public List<Shape> shapes;

  public ShapeAreaCalculator(List<Shape> shapes) {
    this.shapes = shapes;
  }

  public double sumOfShapeAreas() {
    int sum = 0;
    if (shapes.getClass().equals(Rectangle.class)) {
      sum += ((Rectangle) shape).getHeight()
        * ((Rectangle) shape).getWidth();
    }
    return sum;
  }
}
```

계산기 내부에서 각각을 계산한다고 가정합니다. 그렇다면 다른 개발자가 Circle 클래스를 새롭게 구현하고자 하면 어떨까요? 그럼 면적 계산기의 `sumOfShapeAreas()` 메서드에 원의 면적
계산로직을 끼워넣어야하는 수정이 발생합니다.

```java
//...
public double sumOfShapeAreas() {
  int sum = 0;
  if (shapes.getClass().equals(Rectangle.class)) {
    sum += ((Rectangle) shape).getHeight()
      * ((Rectangle) shape).getWidth();
  } else if (shapes.getClass().equals(Circle.class)) { // 수정이 발생하는 부분
    // ...
  }
  return sum;
}
//...
```

#### OCP를 따른 경우

각 객체 면적의 계산은 객체가 처리하도록 수정합니다.

```java
public interface Shape {
  double area();
}
```

원과 사각형 클래스 내부에 `area()`를 구현합니다.

```java
public class Rectangle {
  public double area() {
    // ...
  }
}

public class Circle {
  public double area() {
    // ...
  }
}
```

```java
public class ShapeAreaCalculator {
  public List<Shape> shapes;

  public ShapeAreaCalculator(List<Shape> shapes) {
    this.shapes = shapes;
  }

  public double sumOfShapeAreas() {
    int sum = 0;
    for (Shape shape : shapes) {
      sum += shape.area();
    }
    return sum;
  }
}
```

이렇게 각 객체에게 작업을 할당하면 면적 계산기에서는 단지 클래스의 `area()`메서드만을 이용하면 됩니다. 다른 개발자가 삼각형을 구현해도 `Calculator`의 수정은 발생하지 않으므로 OCP원칙을 지킵니다.

## 리스코프 치환 원칙 (Liskov Substitution Principle, LSP)

### 요점

- 서브클래스의 객체는 슈퍼클래스의 객체와 반드시 같은 방식으로 동작해야 한다.
- 파생 타입은 반드시 기본 타입을 완벽히 대체할 수 있어야 한다.

### 예시

**프리미엄,VIP,무료회원과 바둑동호회**를 예로 들겠습니다.

#### LSP를 따르지 않은 경우

`Member`클래스는 바둑동호회 구성원 클래스입니다.

```java
public abstract class Member {
  private final String name;

  public abstract void joinTournament();

  public abstract void organizeTournament();
  // constructor, getter, setter
} 
```

프리미엄 멤버와 VIP 멤버는 대회를 주최하거나, 참여할 수 있습니다. 반면 무료회원은 대회에 오직 참여만 가능합니다. 주최는 할 수 없습니다.

```java
public class PremiumMember extends Member {
  public PremiumMember(String name) {
    super(name);
  }

  @Override
  public void joinTournament() {
    System.out.println("Premium member joins tournament..");
  }

  @Override
  public void organizeTournament() {
    System.out.println("Premium member organize tournament..");
  }
}
```

```java
public class FreeMember extends Member {
  public FreeMember(String name) {
    super(name);
  }

  @Override
  public void joinTournament() {
    System.out.println("Free member joins tournament..");
  }

  @Override
  public void organizeTournament() {
    throw new IllegalStateException("Free member can't organize tournament");
  }
}
```

이 멤버를 사용하는 곳에서 아래와 같이 코드를 작성하면 어떻게 될까요?

```java
public static void main(String[] args) {
  List<Member> members = List.of(new PremiumMember("김프리미엄"),
    new VipMember("홍브압"),
    new FreeMember("박무료"));
  for (Member member : members) {
    member.organizeTournament();
  }
}
```

반복문의 마지막 박무료의 메서드를 호출하려 할 때 에러가 발생합니다. `FreeMember`가 `Member`를 대체할 수 없기 때문입니다. 이것은 리스코프 치환 원칙에 어긋납니다.

#### LSP를 따른 경우

대회에 참여, 대회를 주최하는 두 가지 일을 분리하는 것으로 시작합니다.

```java
public interface TournamentJoiner {
  void joinTournament();
}

public interface TournamentOrganizer {
  void organizeTournament();
}
```

```java
public abstract class Member implements TournamentJoiner, TournamentOrganizer {
  private final String name;
  // constructor
}

public class PremiumMember extends Member {
}

public class VipMember extends Member {
}

public class FreeMember implements TournamentJoiner { // TournamentJoiner만을 구현한 새로운 FreeMember
}
```

```java
public static void main(String[] args) {
  List<TournamentJoiner> members = List.of(new PremiumMember("김프리미엄"),
    new VipMember("홍브압"),
    new FreeMember("박무료"));

  List<TournamentOrganizer> members = List.of(new PremiumMember("김프리미엄"),
    new VipMember("홍브압"));
  // FreeMember는 TournamentOrganizer를 구현하지 않기 리스트에 포함될 수 없음
}
```

각 리스트를 반복하며 메서드를 실행하면 기대한 방식대로 동작하고, 리스코프 치환 원칙도 준수하는 것을 확인할 수 있습니다.

## 인터페이스 분리 원칙 (Interface Segregation Principle, ISP)

### 요점

- 클라이언트가 사용하지 않을 불필요한 메서드를 강제로 구현하게 해서는 안된다.
- 메서드를 강제로 구현하는 일이 없을 때까지 하나의 인터페이스를 2개 이상의 인터페이스로 분할한다.

### 예시

**Connection**인터페이스를 예로 들겠습니다.

#### ISP를 따르지 않은 경우

```java
public interface Connection {
  void socket();

  void http();

  void connect();
}
```

`Connection` 인터페이스를 구현하는 `WwwPingConnection`이 있다고 가정합니다.

```java
public class WwwPingConnection implements Connection {
  @Override
  public void http() {
    // do http..
  }

  @Override
  public void connect() {
    // do connect..
  }

  @Override
  public void socket() {
    // do nothing
  }
}
```

`socket()` 메서드를 사용하지않지만 강제로 구현해야 합니다. 또한 `WwwPingConnection`사용하는 클라이언트는 `socket()`이 아무것도 하지않는 메서드인지 모릅니다. 이는 ISP
원칙에 위배됩니다.

#### ISP를 따른 경우

```java
public interface Connection {
  void connect();
}

public interface HttpConnection extends Connection {
  void http();
}

public interface SocketConnection extends Connection {
  void socket();
} 
```

인터페이스를 분리한 후 `WwwPingConnection`를 다시 구현합니다.

```java
public class WwwPingConnection implements HttpConnection {
  @Override
  public void http() {
    // do http..
  }

  @Override
  public void connect() {
    // do connect..
  }
}
```

`WwwPingConnection`은 필요한 메서드만 구현합니다. 이는 ISP 원칙을 준수합니다.

## 의존관계 역전 원칙 (Dependency Inversion Principle, DIP)

### 요점

- 구체화가 아닌 추상화에 의존해야한다.
- 다른 구상 모듈에 의존하는 구상 모듈 대신, 구상 모듈을 결합하기 위한 추상 계층을 사용한다.

### 예시

**데이터베이스 JDBC URL**클래스를 예로 들겠습니다.

#### DIP를 따르지 않은 경우

```java
public class PostgresSQLJdbcUrl {
  private final String dbName;

  // constructor
  public String get() {
    return "jdbc:postgresql://.." + dbName;
  }
}
```

```java
public class ConnectToDatabase {
  public void connect(PostgresSQLJdbcUrl postgresql) {
    System.out.println("Connecting to " + postgresql.get());
  }
}
```

MySQLJdbcUrl과 같이 다른 JDBC URL 타입을 생성하는 경우엔 ConnectToDatabase.connect()를 사용할 수 없습니다. 이럴 때 구체화가 아닌 추상화에 대한 의존관계를 만들어야 합니다.

#### DIP를 따르는 경우

```java
public interface JdbcUrl {
  String get();
}
```
```java
public class PostgresSQLJdbcUrl implements JdbcUrl {
  private final String dbName;

  // constructor
  @Override
  public String get() {
    return "jdbc:postgresql://.." + dbName;
  }
}
```
```java
public class MySQLJdbcUrl implements JdbcUrl {
  private final String dbName;

  // constructor
  @Override
  public String get() {
    return "jdbc:mysql://.." + dbName;
  }
}
```
```java
public class ConnectToDatabase {
  public void connect(JdbcUrl jdbcUrl) {
    System.out.println("Connecting to " + jdbcUrl.get());
  }
}
```

여러 벤더사에 대한 추상화를 통해 JdbcUrl 인터페이스를 생성합니다. `ConnectToDatabase`에서 추상화 한 JdbcUrl 인터페이스에 의존합니다. 이 후 JdbcUrl을 구현한 클래스라면 `ConnectToDatabase`에서 사용할 수 있습니다. 이제 DIP 원칙을 만족합니다.  

## 마치며

SOLID에 대한 개념뿐만 아니라 구체화된 예시를 보며 이해도를 높일 수 있었습니다.

읽어주셔서 감사합니다.

## reference
> [자바 코딩 인터뷰 완벽 가이드 - 안겔 레오나르드](https://www.yes24.com/Product/Goods/111393077?pid=123487&cosemkid=go16600967206182417&gad_source=1&gclid=CjwKCAiAuYuvBhApEiwAzq_YiUi-pJoDhKWmQKj4ANhCNBTeG_sFDvaFtbtTk3OC37UYJeKzqm5bKRoCdfsQAvD_BwE)

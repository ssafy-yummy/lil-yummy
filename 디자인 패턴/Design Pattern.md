# Design Pattern

# 1. 디자인 패턴이요??

![Untitled](Design%20Pattern%20003472965ef64c85a1d92292be5fd839/Untitled.png)

코드를 작성하다 보면 중복된 코드로 인해 더러워지는 경우를 볼 수 있다.

뿐만아니라 그런 코드를 줄일 생각 없이 ctrl c + v하고있는 자신을 보는 경우도 허다하다.

**보기 좋은 떡이 먹기도 좋다하지 않는가,**

이처럼 공통된 코드로 인해 발생하는 문제를 해결하기 위해 
”**Using Pattern Languages for Object-Oriented Programs”** 에서 제안되고, 정리되고  보편화된 
개발에 대한 방향성 그것이 `**Design Pattern**`이다.

<aside>
👀 디자인 패턴(Design Pattern)은 소프트웨어 공학의 소프트웨어 설계에서 공통으로 발생하는 
문제에 대해 자주 쓰이는 설계 방법을 정리한 패턴이다.

</aside>

디자인 패턴의 장점은 다음과 같다.

- 설계에 대한 추상화를 통해 재사용성을 높인다.
- 설계 추상화를 통한 변경의 유연함을 보인다.

여기서는 `**Design Pattern**` 중 GoF(Gang of Four)가 작성한 “Design Patterns“를 통해 
23가지의 디자인 패턴 중 일부를 다룬다.

# 2. GoF Design Patterns

23가지의 디자인 패턴은 총 3가지의 카테고리로 분류된다.

각각 `생성`, `구조`, `행위` 로 구분된다.

구분되는 내용들은 다음과 같다.

| 생성 | 구조 | 행위 |
| --- | --- | --- |
| - 추상 팩토리
- 빌더
- 팩토리 메서드
- 프로토 타입
- 싱글톤 | - 어댑터
- 브리지
- 컴포지트
- 데코레이터
- 퍼사드
- 플라이웨이트
- 프록시 | - 책임 연쇄
- 커맨드
- 인터프리터
- 이터레이터
- 미디에이터
- 메멘토
- 옵저버
- 상태
- 전략
- 템플릿 메서드
- 비지터 |

## 1. 생성 패턴

생성 패턴은 객체 인스턴스를 생성하는 패턴으로 어떤 구체화 클래스를 사용하는지에 대해 
은닉/캡슐화하는 역할을 한다.

생성패턴을 거친다면 객체가 생성되거나 변경되어도 프로그램 구조에 영향을 받지 않는다.

### 1. 싱글톤 패턴(Singleton Pattern)

싱글톤 패턴은 객체 인스턴스가 하나만 생성되도록 하는 패턴이다.

전역변수와 마찬가지로 객체 인스턴스가 어디든 접근할 수 있도록 한다.

장점

- 메모리 낭비 방지
- 이미 생성된 인스턴스를 활용한 속도
- 전역으로 구성된인스턴스, 따라서 다른 클래스의 인스턴스들이 데이터 공유 가능

단점

- 단일 객체가 많은 역할을 해야하는 만큼 
공유되는 데이터가 증가하면 다른 클래스들 간의 결합도가 높아진다.

구현 방식은 다음과 같다.

```java
Public Class Singleton{

	private Singleton(){} // private를 통한 외부생성 제한

	private static class createSingleton{ // jvm을 통한 싱글톤의 초기화 제한
		public static final Singleton instance = new Singleton();
	}
	
	public static Singleton getInstance(){ // 객체를 가져온다.
		return createSingleton.instance;
	}
}
```

### 2. 팩토리 메서드

객체를 만드는 과정을 Sub class에 위임하는 패턴

부모 추상 클래스는 인터페이스에만 의존하며 실질적인 구현은 서브 클래스에서 구현한다.

이때 새로운 구현 클래스가 추가되어도 기존 Fatory 코드의 수정 없이 새로운 Factory를 추가하면 된다.

장점

- SOLID 에서 O(open closed principle)을 지킬 수 있다.

단점

- 기능 구현에 있어 많은 클래스를 정의해야하기 때문에 코드량이 증가한다.
1. user interface 정의

```jsx
public interface User{
	void hello();
}
```

1. user interface를 상속받는 UserImpl

```jsx
public class FirstUser implements User{
	@Override
	public void hello(){
		System.out.println("hello");
	}
}
```

1. UserFactory

추상 클래스 UserFactory

외부에서 User를 생성할 경우 newInstance()메서드로 호출하였고, 실제로 어떤 객체를 생성할 지는 추상 메서드로 정의해서 하위 클래스에서 정의

```jsx
public abstract class UserFactory {

    public User newInstance() {
        User user = createUser();
        user.hello();
        return user;
    }

    protected abstract User createUser();
}
```

1. UserFactory

```jsx
public class FirstUserFactory extends UserFactory {
    @Override
    protected User createUser() {
        return new FirstUser();
    }
}
```

1. 클라이언트

```jsx
UserFactory userFactory = new FristUserFactory();
User user = userFirstFactory.newInstance();
```

만약 여기서 객체를 추가한다고 가정한다면

```jsx
public class SecondUser implements User{
	@Override
	public void hello(){
		System.out.println("Hello2");
	}
}
```

User를 생성하기 위한 팩토리 클래스를 정의한다면 다음과 같다.

```jsx
public class SecondUserFactory extends UserFactory {
    @Override
    protected User createUser() {
        return new SecondUser();
    }
}
```

### 3. 추상 팩토리 패턴

관련있는 객체들을 묶어 하나의 팩토리 클래스로 만들고 
이를 조건에 따라 생성하는 팩토리를 만들어 객체를 생성하는 패턴

![Untitled](Design%20Pattern%20003472965ef64c85a1d92292be5fd839/Untitled%201.png)

```jsx
public class Ship {
  private Anchor anchor;
  private Wheel wheel;

  public Ship() {
  }

  public Anchor getAnchor() {
    return anchor;
  }

  public void setAnchor(Anchor anchor) {
    this.anchor = anchor;
  }

  public Wheel getWheel() {
    return wheel;
  }

  public void setWheel(Wheel wheel) {
    this.wheel = wheel;
  }
}

public class WhiteShip extends Ship{
    public WhiteShip(){}
}
```

```jsx
public interface ShipFactory{
    Ship createShip();
}

public class WhiteShipFactory implements ShipFactory{
    @Override 
    public Ship createShip(){
        Ship ship = new WhiteShip();
        ship.setAnchor(new WhiteAnchor());
        ship.setWheel(new WhiteWheel());
    }
}
```

```jsx
public class Client{
  public static void main(String[] args) {
    ShipFactory whiteShipFactory = new WhiteShipFactory();
    Ship whiteShip = whiteShipFactory.createShip();
  }
}
```

변경 후

```jsx
public interface ShipPartsFactory {
  Anchor createAnchor();

  Wheel createWheel();
}
public class WhiteSheepPartsFactory implements ShipPartsFactory {

  @Override
  public Anchor createAnchor() {
    return new WhiteAnchor();
  }

  @Override
  public Wheel createWheel() {
    return new WhiteWheel();
  }
}
```

```jsx
public class WhiteShipFactory implements ShipFactory{
    private ShipPartsFactory shipPartsFactory;

    public WhiteShipFactory(ShipPartsFactory shipPartsFactory) {
        this.shipPartsFactory = shipPartsFactory;
    }

    @Override
    public Ship createShip() {
        Ship ship = new WhiteShip();
        ship.setAnchor(shipPartsFactory.createAnchor());
        ship.setWheel(shipPartsFactory.createWheel());

        return ship;
    }
}

public class Client {
  public static void main(String[] args) {
    ShipFactory whiteShipFactory = new WhiteShipFactory(new WhiteSheepPartsFactory());
    Ship whiteShip = whiteShipFactory.createShip();

    ShipFactory whiteShipProFactory = new WhiteShipFactory(new WhiteSheepProPartsFactory());
    Ship whiteShipProship = whiteShipProFactory.createShip();
  }
}
```

### 4. 프로토 타입

![Untitled](Design%20Pattern%20003472965ef64c85a1d92292be5fd839/Untitled%202.png)

- 인스턴스를 복사하는 인터페이스인 Prototype을 선언하고 이를 구현하는 구현 클래스를 생성한다.
인터페이스에는 Clone()메서드를 선언
- 구현 클래스에서는

```java
package designpattern.prototype;

public interface Prototype {
    public Object clone() throws CloneNotSupportedException;
}
```

prototype 구현 클래스

```java
package designpattern.prototype;

import java.util.ArrayList;
import java.util.List;

public class User implements Prototype{
    private List<String> usersList;

    public User() {
        usersList = new ArrayList<>();
    }

    public User(List<String> usersList) {
        this.usersList = usersList;
    }

    public List<String> getUsersList() {
        return usersList;
    }

    public void ListAll(){
        usersList.add("a");
        usersList.add("b");
        usersList.add("c");
    }

    @Override
    public Object clone() throws CloneNotSupportedException {
        return new User(new ArrayList<>(this.usersList));
    }
}
```

클라이언트

```java
package designpattern.prototype;

import java.util.Arrays;
import java.util.List;

public class Main {
    public static void main(String[] args) throws CloneNotSupportedException {
        User user = new User();
        user.ListAll();

        // 프로토타입 구현체 복사
        User user1 = (User) user.clone();
        User user2 = (User) user.clone();

        List<String> userList1 = user1.getUsersList();
        List<String> userList2 = user2.getUsersList();

        userList1.add("d");
        userList2.add("e");

        System.out.println(Arrays.toString(user.getUsersList().toArray()));
        System.out.println(Arrays.toString(userList1.toArray()));
        System.out.println(Arrays.toString(userList2.toArray()));

    }
}
```

![Untitled](Design%20Pattern%20003472965ef64c85a1d92292be5fd839/Untitled%203.png)

프로토타입 패턴의 적용하는 예시는 다음과 같다.

DB혹은 HTTP 통신을 통해 데이터를 가져올 경우 비용 부분을 최소화하기 위해사용

## 2. 구조 패턴

구조패턴은 클래스 혹은 객체를 조합해 더큰 하나의 구조를 만드는 패턴이다.

이때 특징은 다음과 같다.

- 독립적으로 동작하는 복수의 기능을 하나의 인터페이스 내에서 정의
- 컴파일 단계가 아닌 런타임 단계에서 조합방법이나 대상을 변경할 수 있다는 점에서 
유연성을 갖는다.

### 2.1. 퍼사드 패턴

![Untitled](Design%20Pattern%20003472965ef64c85a1d92292be5fd839/Untitled%204.png)

- Facade
    - 사용자의 요청을 SubSystem으로 전달하는 단순하고 통합된 인터페이스
- SubSystem Class
    - Facade에 대한 정보를 갖지 않고, 서브 시스템의 기능을 구현하는 클래스

예를 들어보자.

각각 test 1, 2, 3을 출력하는 세개의 서브 시스템 클래스가 있다.

(말이 그렇다는 거지 그냥 클래스이다.)

```jsx
public class Test1 {
    public void testPrint1(){
        System.out.println("Facade test1");
    }
}
/*-----------------------------------------*/
public class Test2 {
    public void testPrint2(){
        System.out.println("Facade test2");
    }
}
/*-----------------------------------------*/
public class Test3 {
    public void testPrint3(){
        System.out.println("Facade test3");
    }
}
```

정의한 서브 시스템 클래스를 하나의 클래스에 담는다.

```jsx
public class Facade {

    public void  createFacade(){
        Test1 test1 = new Test1();
        Test2 test2 = new Test2();
        Test3 test3 = new Test3();

        test1.testPrint1();
        test2.testPrint2();
        test3.testPrint3();
    }

}
```

Main에서 이를 호출하기만 하면 우리는 하위 서브 시스템 클래스를 신경쓰지 않아도 된다.

```jsx
public class Main {
    public static void main(String[] args) {
        Facade facade = new Facade();
        facade.createFacade();
    }
}
```

이때의 특징은 다음과 같다.

1. 클라이언트가 서브클래스를 모르더라도 사용할 수 있다.
2. 파서드 클래스를 호출하지 않더라도 서브클래스를 직접 호출할 수 있다.

### 2.2.  프록시 패턴

![Untitled](Design%20Pattern%20003472965ef64c85a1d92292be5fd839/Untitled%205.png)

프록시 패턴은 실제 기능을 수행하는 객체의 접근을 제어하거나 
원래 기능의 앞 혹은 뒤에 추가기능을 더하는 것이다.

전처리 혹은 후처리에 용이하다(중간에서 )

공통 인터페이스

```jsx
package proxy;

public interface Subject {
    void testSub();
}
```

```jsx
package proxy;

public class RealSubject implements Subject{

    public RealSubject() {
        System.out.println("create RealSubject");
    }

    @Override
    public void testSub() {
        System.out.println("test");
    }
}
```

```jsx
package proxy;

public class ProxySubject implements Subject{

    private Subject subject;

    @Override
    public void testSub() {
        if(subject==null){
            subject = new RealSubject();
        }
    }
}
```

```jsx
package proxy;

public class Main {
    public static void main(String[] args) {
        Subject proxy = new ProxySubject();
        proxy.testSub();
    }
}
```

## 3. 행위 패턴

객체간의 상호작용이나 책임 방법을 정의하는 패턴

### 3.1. 전략 패턴

![Untitled](Design%20Pattern%20003472965ef64c85a1d92292be5fd839/Untitled%206.png)

같은 문제를 해결하기 위한 다양한 메서드를 클래스 별로 캡슐화시키고 필요 시 이를 교체해가며 

적용시킬 수 있도록 하는 디자인 패턴

 

```java
public abstract class Robot{
    private String name;
    public Robot(String name){
        this.name = name;
    }
    
    public String getName(){
        return name;
    }
    
    public abstract void attack();
    public abstract void move();
}
```

위와 같은 상황에서 동작이 추가되거나 수정되어야 하는 경우 코드의 길이가 복잡해진다는 단점이 있다.

```java
public abstract class Robot{
    private String name;
    private MovingStrategy movingStrategy;
    private AttackStrategy attackStrategy;
    
    public Robot(String name){
        this.name = name;
    }
    
    public String getName(){
        return name;
    }
    
    public void move(){
        movingStrategy.move();
    }
    
    public void attack(){
        attackStrategy.attack();
    }
    
    public void setMovingStrategy(MovingStrategy movingStrategy){
        this.movingStrategy = movingStrategy;
    }
    
    public void setAttackStrategy(AttackStrategy attackStrategy){
        this.attackStrategy = attackStrategy;
    }
}
```
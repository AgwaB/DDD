## Domain Driven Development

아래의 개념은 전혀 어렵지도 않고 이때까지 많이 봐왔던 내용들이다. 이 내용을 DDD의 관점 즉, 도메인의 관점에서 바라보는 것이 중요하다!

### 도메인 모델
 특정 도메인을 개념적으로 표현한 것, 아키텍처상의 도메인 계층을 객체 지향 기법으로 구현하는 패턴

- 개념 모델은 순수하게 문제를 분석한 결과물이며 , db, 트랜잭션, 성능, 구현 등을 고려하지 않는다. 이후에 구현 가능한 형태의 모델로 전환하는 과정을 거치게 된다. 
- 프로젝트를 하면서 도메인에 대한 이해도가 달라져서 프로젝트 초기의 도메인과 후반부의 도메인은 달라질 수 있다. 
- 즉 초기에는 개요 수준의 개념 모델로 도메인에 대한 전체 윤곽을 이해하는데 집중하고, 구현하는 과정에서 개념 모델을 구현 모델로 점진적으로 발전시켜 나가야 한다.

> 결론 : 완벽한 초기설계는 불가능!


### Entity와 Value

도출한 모델은 크게 엔티티와 밸류로 구분할 수 있다.

#### Entity
식별자를 가진다는게 가장 큰 특징이다. 엔티티를 생성하고 엔티티의 속성을 바꾸고 엔티티를 삭제할 때까지 식별자는 유지된다.

식별자는 

- 특정 규칙에 따라 생성
- UUID 사용
- 값을 직접 입력
- 일련번호(시퀀스나 DB의 자동 증가 칼럼) 사용

등을 이용해 생성할 수 있다.

> MySQL같은 경우에는 auto increment가 있다. 하지만 이건 DB에 넣은 이후에 생성되는 것임을 주의

> 식별자를 이용해서 `equals()` 와 `hashCode()` 메서드를 오버라이드 해주자.


#### Value

```java
public class ShppingInfo {
    private String receiverName;
    private String receiverPhoneNumber;
    
    private String shippingAddress1;
    private String shippingAddress2;
    private String shippingZipcode;
    
    ...
}
```

가 있으면 위의 `receiverName`과 `receiverPhoneNumber`는 개념적으로 받는 사람을 의미한다.

또한 `shippingAddress1`, `shippingAddress2`, `shippingZipcode`는 주소라는 하나의 개념을 표현한다.

위의 두가지 개념은 Value 타입을 사용함으로써 개념적으로 완전한 하나를 표현할 수 있다.
```java
public class Receiver {
    private String name;
    private String phoneNumber;
    
    public Receiver(String name, String phoneNumber) {
        this.name = name;
        this.phoneNumber = phoneNumber;
    }
    
    public String getName() {
        return name;
    }      
    
    public String getPhoneNumber() {
        return phoneNumber;
    }   
}
```

```java
public class Address {
    private String address1;
    private String address2;
    private String zipcode;
    
    public Address(String address1, String address2, String zipcode) {
        ...
    }
    ...
}
```

와 같이 묶을 수 있다.

그리고 
```java
public class OrderLine {
    private Product product;
    private int price;
    private int quantity;
    private int amounts;
    ...
}
```

에서 `price`와 `amounts`는 int 타입의 숫자지만 의미하는 값은 `돈`이다. 즉 `돈`을 의미하는 `Money` 타입을 만들어서 사용하면 코드를 이해하는 도움이 된다.
```java
public class Money {
    private int value;
    
    public Money(int value) {
        this.value = value;
    }
    
    public int getValue() {
        return this.value;
    }
}
```

```java
public class OrderLine {
    private Product product;
    private Money price;
    private int quantity;
    private Money amounts;
}
```

이렇게 Value타입을 이용하면 가독성 향상이 가능하다.


#### Immutable

- 보다 안전한 코드

```java
Money price = new Money(1000);
OrderLine line = new OrderLine(product, price, 2);
price.setValue(2000);
```

이면 price가 중간에 변경된다. 이것을 피하기 위해 OrderLine 안의 생성자에서 데이터를 복사 한 후(객체 재생성) 넣어주는데 너무 비효율적임..

> 객체 내 변수를 `final`을 사용해서 명시적으로 Immutable하게 만들어 주자.


##### 도메인 모델에는 set 메서드를 넣지 말자

>  set 메서드는 도메인의 핵심 개념이나 의도를 코드에서 사라지게 한다!!

```java
// set 메서드로 데이터를 전달하면?
// 처음 Order를 생성하는 시점에 order는 완전하지 않다!
Order order = new Order();

// set 메서드로 필요한 모든 값을 전달해야 한다. 이는 Mutable하기도 하다.
order.setOrderLine(lines);
order.setShippingInfo(shippingInfo);
```

위와 같이 `도메인 객체`가 불완전한 상태로 사용되는 것을 마긍려면 생성 시점에 필요한 것을 전달해 주어야 한다. 즉, `생성자를 통해 필요한 데이터를 모두 받아야 한다`.


##### Naming

코드를 작성할 때 도메인에서 사용하는 용어는 매우 중요하다. 도메인에서 사용하는 용어를 코드에 반영ㄹ하지 않으면 그 코드를 개발자에게 코드의 의미를 해석해야 하는 부담을 준다.

> 도메인 용어를 사용해라

- setOrderState()와 같은 메서드는 상태 값만 변경할지, 상태 값에 따라 다른 처리를 위한 코드를 함께 구현할지 애매하다.

- completePayment()와 같은 메서드는 결제 완료와 관련된 처리 코드를 함께 구현하기 때문에 결제 완료와 관련된 `도메인 지식을 코드로 구현하는 것이 자연스럽다`.


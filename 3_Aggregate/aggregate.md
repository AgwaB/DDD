# Aggregate

- 프로젝트가 커지다보면 엔티티와 밸류가 많아짐에 따라 관리하기가 어렵고 도메인 상위 수준에서 모델을 조망할 수 있어야 한다.

- 이 때, 수많은 객체를 애그리거트로 묶어서 바라보면 좀 더 상위 수준에서 도메인 모델 간의 관계를 파악할 수 있다.

- 한 애그리거트에 속한 객체는 유사하거나 동일한 라이프사이클을 갖는다.

- 경계를 설정할 때 기본이 되는 것은 `도메인 규칙`과 `요구사항`이다.

- OrderLine의 주문 상품 개수를 변경하면 도메인 규칙에 따라 Order의 총 주문 금액을 새로 계산해야 한다. 이런식으로 `함께 변경되는 빈도가 높은 객체는 한 애그리거트에 속할 가능성이 높다`.

- 'A가 B를 갖는다'로 설계할 수 있는 요구사항이 있으면 A와 B를 한 애그리거트로 묶어서 생각하기 쉽지만 그렇지 않은 경우도 있다. (Product, Review 객체의 경우 Product는 Review를 가지지만 주체가 다르다.)


 ### 애그리거트 루트
 
 애그리거트에 속한 모든 객체가 일관된 상태를 유지하려면 애그리거트 전체를 관리할 주체가 필요하다. 이 책임을 지는 것이 애그리거트 루트 엔티티이다.
 
 루트의 역할

- 애그리거트의 일관성이 깨지지 않도록 하는 것.

일관성이 깨지지 않게하는 가장 쉬운 방법?

- set 메서드를 public으로 하지 않는다.
- 밸류 타입은 불변으로 구현한다.

왜? 

- `set 메서드를 사용하지 않으면 의미가 드러나는 메서드를 사용해서 구현할 가능성이 높아진다!`

```java
public class Order {
    private ShippingInfo shippingInfo;
    public void changeShippingInfo(ShippingInfo newShippingInfo) {
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo);
    }
    
    // private이다.
    private void setShippingInfo(ShippingInfo newShippingInfo) {
        // 새로운 객체로 교체한다. (Immutable이기 때문)
        this.shippingInfo = newShippingInfo;
    }
}
```

이렇게 `밸류타입의 내부 상태를 변경하려면 애그리거트 루트를 통해서만 가능하게 한다`. 애그리거트 루트가 도메인 규칙을 올바르게만 구현하면 애그리거트 전체의 일관성을 올바르게 유지할 수 있다.


### 애그리거트 루트의 기능 구현

애그리거트 루트는 애그리거트 내부의 다른 객체를 조합해서 기능을 완성한다.

```java
public class Order {
    private Money totalAmouints;
    private List<OrderLine> orderLines;
    
    private void calculateTotalAmounts() {
        ...
    }
    ...
}
```

에서 OrderLine에 기능 실행이 Order에 있다. 이건 OrderLine에게 기능 실행을 `위임`하기도 한다,. 

```java
public class OrderLines {
    private List<OrderLine> lines;
    
    public int getTotalAmounts() { ... }
    public void changeOrderLines(List<OrderLine> newLine) {
        this.lines = newLines;
    }
}
```

```java
public class Order {
    private OrderLines orderLines;
    
    public void changeOrderLines(List<OrderLine> newLines) {
        orderLines.changeOrderLines(newLines);
        this.totalAmounts = orderLines.getTotalAmounts();
    }
}
```

이러면 애그리거트 외부에서 OrderLines의 기능을 실행할 수 있게 된다.

아직은 필요성을 크게 못느끼겠는데 프로젝트 하다보면 마딱드릴 수 있을 것 같다.

> `OrderLines`는 Immutable하게 구현하라! 불변으로 구현 할 수 없다면 protected로 범위를 한정해서 패키지 외부에서 실행할 수 없도록 제한하라!


### 트랜잭션 범위

 - 한 트랜잭션에는 한 개의 애그리거트만 수정해야 한다. 최대한 독립적이어야 한다. : `애그리거트에서 다른 애그리거트를 변경하지 않아야 한다`.
 - 왜? 애그리거트 간 의존하면 결합도가 높아지게되고 향후 수정 비용이 증가한다.
 
 애그리거트 안에서 다른 애그리거트를 건드리는 경우
 ```java
public class Order {
    private Orderer orderer;
    
    public void shipTo(ShippingInfo newShippingInfo, boolean useNewShippingAddrAsMemberAddr) {
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo);
        if (useNewShippingAddrAsMemberAddr) {
            // 다른 애그리거트의 상태를 변경하면 안된다!
            ordere.getCustomer().changeAddress(newShippingInfo.getAddress());
        }
    }
}
```


* 근데 어쩔 수 없이(요구사항이 이렇게 온다거나?) 두 개 이상의 애그리거트를 수정해야한다면 애그리거트 안에서 직접 수정하는 것이 아니라 Application Service에서 두 애그리거트를 수정하도록 구현해야 한다.

```java
public class ChangeOrderService {
    @Transactional
    public void changeShippingInfo(OrderId id, shippingInfo newShippingInfo, boolean useNewShippingAddrAsMemberAddr) {
        Order order = orderRepository.findbyId(id);
        if (orderr == null) throw new OrderNotFoundException();
        order.shipTo(newShippingInfo);
        if (useNewShippingAddrAsMemberAddr) {
            order.getOrderer().getCustomer().changeAddress(newShippingInfo.getAddress());
        }
    }
    ...
}
```

한 트랜잭션에서 두 개 이상의 애그리거트를 변경하는 것을 고려할 수 있을때?

- 팀 표준
- 기술 제약
- UI 구현의 편리


### 리포지터리와 애그리거트


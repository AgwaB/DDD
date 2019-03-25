# Architecture

- 표현(Presentation)
- 응용(Application)
- 도메인(Domain)
- 인프라스트럭처(Infrastructure)

보통
웹 <-> Presentation <-> Application 

이런식으로 아키텍쳐가 구성된다. Presentation layer에는 `Controller`가, Application layer에는 `Service`가 보통 위치하게 된다.

Application layer는 로직을 직접 수행하기보다는 Domain Model에 로직 수행을 위임한다. 아래의 코드를 보면 주문 취소 `로직을 직접 구현하지 않고` Order 객체에 취소 처리를 `위임하고 있다`.
```java
public class CancelOrderService {
    
    @Transactional
    public void cancelOrder(String orderId) {
        Order order = findOrderById(orderId);
        if (order == null) throw new OrderNotFoundException(orderId);
        order.cancel();
    }
    ...
}
```

즉 `Order`라는 도메인에서 핵심 로직을 구현한다.

> Domain에는 1장에서 공부했듯이 크게 Entity와 Value로 구성된다.

### 계층 구조 아키텍처

`Presentation -> Application -> Domain -> Infrastructure`

이런 계층 구조를 가진다. Layer의 특성상 `상위 계층에서 하위 계층으로의 의존만 존재`하고 `하위 계층은 상위 계층에 의존하지 않는다`.

그리고 구현의 편리함을 위해 계층구조를 유연하게 적용하기도 한다. 근데 이러면 안된대...

> 도메인의 복잡도에 따라서 Application과 Domain을 합치기도 한다.

Application이나 Domain이 Infrastucture(DB 또는 룰 엔진 같은 상세 구현 기술) 기능을 사용하면(`의존하면`) 어쩔 수 없이 Infrastucture layer에 종속적이게 된다. 이것은 두 가지 문제점이 있다.

- Test가 어렵다.
- 구현 방식을 변경하기 어렵다.

예시
```java
public class CalculateDiscountService {
    private DroolsRuleEngine ruleEngine;
    
    public CalculateDiscountService() {
        ruleEngine = new DroolsRuleEngine();
    }
    
    public Money calculateDiscount(OrderLine orderLines, String customerId) {
        Customer customer = findCustomer(customerId);
        
        MutableMoney money = new MutableMoney(0);
        List<?> facts = Arrays.asList(customer, money);
        facts.addAll(orderLines);
        ruleEngine.ecalute("discountCalculation", facts);
        return money.toImmutableMoney();
    }
    ...
}
```

위의 코드는 `DroolsRuleEngine`에 의존하고 있다. 즉
- 위의 서비스를 테스트 하려면 RuleEngine이 완벽하게 동작해야하기 때문에 관련 설정 파일을 모두 만족 시킨 후 테스트를 할 수 있다.
- "discountCalculation"은 Drools의 세션 이름이다. 이게 변경되면 코드 또한 변경되어야 하고, MutableMoney등 추가작업이 있다.

그래서 나온 개념이 DIP(Dependency Inversion Principle)이다.

### DIP(Dependency Inversion Principle)

고수준 모듈(Service...)의 기능을 구현하려면 여러 하위 기능이 필요하다. 그리고 저수준 모듈(DB, 룰 엔진...)은 하위 기능을 실제로 구현한 것이다.

고수준 모듈은 저수준 모듈에 의존하였는데 이러면 앞서 말한 Test와 구현 변경이 어렵다는 문제가 있었다.

DIP는 의존 관계를 `고수준 모듈 -> 저수준 모듈`에서 `저수준 모듈 -> 고수준 모듈`로 바꾸는 개념이다. 이것은 `인터페이스`로 달성 가능하다!

위의 `CalculateDiscountService`입장에서 봤을 때, 룰 적용을 Drools로 구현했는지, 자바로 직접 구현했는지는 중요하지 않다. 단지, '고객 정보와 구매 정보에 룰을 적용해서 할인 금액을 구한다'는 것이 중요할 뿐이다.
그리고 이를 추상화한 인터페이스는 다음과 같다.
```java
public interface RuleDiscounter {
    public Money applyRules(Customer customer, List<OrderLine> orderLines);
}
```

위의 인터페이스를 이용해서 `CalculateDiscountService`에 적용하면 아래와 같다.
```java
public class CalculateDiscountService {
    private RuleDiscounter ruleDiscounter;
    
    public CalculateDiscountService(RuleDiscounter ruleDiscounter) {
        this.ruleDiscounter = ruleDiscounter;
    }
    
    public Money calculateDiscount(OrderLine orderLines, String customerId) {
        Customer customer = findCustomer(customerId);
        return ruleDiscounter.applyRules(custoemr, orderLines);
    }
    ...
}
```

Drools에 의존하는 것이 아니라 RuleDiscounter가 룰을 적용한다는 것만 안다. 그리고 실제 구현체는 생성자를 통해 받게된다.

그리고 Drools은 RuleDiscounter를 상속 받아서 해당 메소드를 구현한다.
```java
public class DroolsRuleDiscounter implements RuleDiscounter {
    ...
    
    @Override
    public Money calculateDiscount(Customer customer, List<OrderLine> orderLines) {
        ...
    }
    ...
}
```

이러면 `CalculateDiscountService`는 더 이상 Drools에 의존하지 않고 `RuleDiscounter`인터페이스에 의존한다. 또한 `DroolsRuleDiscounter`도 `RuleDiscounter`에 의존한다.

저수준 : `CalculateDiscountService`, `RuleDiscounter`

고수준 : `DroolsRuleDiscounter`

그래서 코드를 사용할 때에는
```java
RuleDiscounter ruleDiscounter = new DroolsRuleDiscounter();

CalculateDiscountService disServie = new CalculateDiscountService(ruleDiscounter);
```

이런식으로 의존주입을 시켜준다. 만약 구현 기술을 변경해야된다면?
```java
RuleDiscounter ruleDiscounter = new SimpleRuleDiscounter();

CalculateDiscountService disServie = new CalculateDiscountService(ruleDiscounter);
```
간단하다.


다음으로 테스트는 어떻게 할까? 코드로 보면...
```java
public class CalculateDiscountService {
    private CustomerRepository customerRepository;
    private RuleDiscounter ruleDiscounter;
    
    public CalculateDiscountService(CustomerRepository customerRepository,
                                    RuleDiscounter ruleDiscounter) {
        this.customerRepository = customerRepository;
        this.ruleDiscounter = ruleDiscounter;
    }
    ...
}
```

이런식으로 주입 받으면 모킹(Mocking)이 유연해진다.
```java
public class CalculateDiscountServiceTest {
    
    @Test(expect = NoCustomerException.class);
    public void noCustomer_thenExceptionShouldBeThrown() {
        CustomerRepository stubRepo = mock(CustomerRepository.class);
        when(stubRepo.findById("noCustId")).thenReturn(null);
        
        RuleDiscounter stubRule = (cust, lines) -> null;
        
        CalculateDiscountService calculateDiscountService =
                new CalculateDiscountService(stubRepo, stubRule);
       ...
    }
}
```

이런식으로 목킹한 객체를 넣어서 테스트 가능하다!

### DIP 주의사항

위에서 언급한 `CalculateDiscountService`와 `RuleEngine`은 고수준 모듈이며 Domain layer이다. 자칫 `RuleEngine`이 저수준 모듈, Infrastucture에 위치하여 인터페이스를 추출하는 경우가 있다. 

### DIP와 아키텍처

추가적인 요구사항이 들어오면 Application이나 Domain의 interface를 구현 하는 클래스를 Infrastucture layer에 추가하면 된다.
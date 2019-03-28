# Presentation layer and Application layer

각 레이어를 지칭하는데 있어서 영어로 다 쓰기 귀찮아서 한글과 영어가 섞여있습니다... 

표현 영역은 `사용자의 요청을 해석`한다. http를 예로 들면 요청을 받은 표현 영역은 URL, 파라미터, 쿠키, 헤더 등을 이용해서 사용자가 어떤 기능을 실행하고 싶어 하는지 판별하고 그 기능을 제공하는 `응용 서비스를 실행`한다.

응용 영역은 실제 사용자가 원하는 기능을 제공하는 영역이다. 회원 가입을 요청하면 표현 영역에서 무슨 요청인지 해석하고 그 기능을 실행하는게 응용 영역이다. 

예시
```java
@RequestMapping(value = "/member/join")
public ModelAndView join(HttpServletRequest request) {
    String email = request.getParameter("email");
    String password = request.getParameter("password");
    // 사용자 요청을 응용 서비스에 맞게 변환
    JoinRequest joinReq = new JoinRequest(email, password);
    // 변환한 객체(데이터)를 이용해서 응용 서비스 실행
    joinService.join(joinReq);
    ...
}
``` 

> 응용 서비스는 표현 영역에 의존하지 않는다! 응용 영역은 사용자가 웹 브라우저를 사용하는지, REST API를 호출하는지., TCP소켓을 사용하는지 여부를 알 필요가 없고, 단지 기능 실행에 대한 필요한 입력값을 전달받고 실행 결과만 리턴하면 될 뿐이다.


### 응용 서비스의 역할

Domain과 Presentation layer를 연결해주는 facade 역할을 한다. 

```java
public Result doSomeFunc(SomeReq req) {
    // 1. 리포지터리에서 애그리거트를 구한다.
    SomeAgg agg = someAggRepository.findById(req.getId());
    checkNull(agg);
    
    // 2. 애그리거트의 도메인 기능을 실행한다.
    agg.doFunc(req.getValue());
    
    // 3. 결과를 리턴한다.
    return createSuccessResult(agg);
}

public Result doSomeCreation(CreateSomeReq req) {
    // 1. 데이터 중복 등 데이터가 유효한지 검사한다.
    checkValid(req);
    
    // 2. 애그리거트를 생성한다.
    SomeAgg newAgg = createSome(req);
    
    // 3. 리포지터리에 애그리거트를 저장한다.
    someAggRepository.save(newAgg);
    
    // 4. 결과를 리턴한다.
    return createSuccessResult(newAgg);
}
```

> 응용 서비스가 이것보다 복잡하다면 응용 서비스에서 도메인 로직의 일부를 구현하고 있을 가능성이 높다!!

`도메인 로직 넣지 않기`

```java
public class ChangePasswordService {
    
    public void changePassword(String memberId, String oldPw, String newPw) {
        Member member = memberRepository.findById(memberId);
        checkMember();
        
        // 얘는 도메인 로직이다!!!
        if (!passwordEncoder.matches(oldPw, member.getPassword())) {
            throw new BadPasswordException();
        }
        
        member.setPassword(newPw);
    }
}
```
-> 실행은 서비스에서 하지만 로직은 도메인에 있어야한다!

**Application 단에 Domain 로직이 들어가면 일어나는 문제점**

- 코드의 응집성이 떨어진다. 도메인 로직을 파악하기 위해 여러 영역을 분석해야 한다.

- 여러 Application Service에 동일한 도메인 로직을 구현할 가능성이 높아진다.

즉 코드의 중복이 발생하고, 코드 변경을 어렵게 만든다. 소프트웨어의 중요한 경쟁 요소 중 하나는 `변경의 용이성`인데 이 가치를 떨어뜨린다.


### *응용 서비스의 구현*

응용 서비스 자체는 `복잡한 로직을 수행하지 않는다`.

회원 도메인을 생각한다면, 보통 두 가지 방식으로 구현한다.

1. 한 응용 서비스 클래스에 회원 도메인의 모든 기능 구현하기
2. 구분되는 기능별로 응용 서비스 클래스를 따로 구현하기

1번처럼 구현하면 `동일한 로직을 위한 코드 중복을 제거하는 것이 쉽지만`, `서비스 클래스가 비대해지고(코드가 길어지고) 연관성이 적은 코드가 한 클래스에 함께 위치할 가능성이 높아진다`.

또한 습관처럼 하나의 클래스에 억지로 끼워넣는 상황이 발생하게 된다.

그래서 2번처럼 하면 `클래스의 개수는 많아지지만`, `코드 품질을 일정 수준으로 유지하는데 도움이 된다`.

하지만 여러 서비스 클래스에서 공통적으로 사용하는 로직이 있을 수 있다 이럴 경우 별도 클래스에 로직을 구현해서 코드가 중복되는 것을 방지할 수 있다.
```java
public final class MemberServiceHelper {
    public static Member findExistingMember(MemberRepository repo, String memberId) {
        Member member = memberRepository.findById(memberId);
        if (member == null) {
            throw new NoMemberException(memberId);
        }
        return member;
    }
    ...
}
```

### 응용 서비스의 인터페이스와 클래스

굳이 필요할까?

인터페이스가 필요한 몇 가지 상황

- 구현 클래스가 다수 존재할 때
- 런타임에 구현 객체를 교체해야 할 경우

> 근데 Application Service는 보통 런타임에 이를 교체하는 경우가 거의 없을 뿐만 아니라 한 Application Service의 구현 클래스가 두 개인 경우도 매우 드물다.

위와 같은 이유로, 인터페이스와 클래스를 따로 구현하면 소스 파일만 많아지고 구현 클래스에 대한 간접 참조가 증가해서 전체 구조만 복잡해진다. 즉, 인터페이스가 명확하게 필요하기 전까지는 쓰지말자!

이유?

- 테스트 할 때도 Mockito쓰면 목킹 다 해주는데... 필요없음


### 파라미터와 값 리턴

스프링 MVC같은 경우 파라미터가 2개 이상 넘어가면 request 파라미터를 자바 객체로 변환해주는 기능을 제공해서 별도 클래스를 사용하는 것이 편리하다. 

> 애그리거트를 리턴해 줄 수도 있지만, Application Service는 Presentation layer에서 필요한 데이터만 리턴하는 것이 기능 실행 로직의 응집도를 높이는 확실한 방법이다.

### Presentation layer에 의존하지 않기

request나 session을 Application Service에 넘기면 둘 사이의 의존성이 생긴다. 표현 영역이 변경되면 응용 서비스의 구현도 함께 변경되어야 하므로 이는 유지보수가 힘들게 만든다. 


### 트랜잭션 처리

프레임 워크에서 제공하는 기능을 적극 이용하자. ( `@Transactional` )
`@Transactional`이 적용된 메소드에서 `RuntimeException`이 발생하면 트랜잭션을 롤백하고 그렇지 않으면 커밋해라.


### 도메인 이벤트 처리

Application Service의 역할 중 하나는 Domain layer에서 발생시킨 이벤트를 처리하는 것이다.
> 여기서의 이벤트란 도메인에서 발생한 상태 변경을 의미한다.

이벤트의 구현에 대해서는 10장에서 다룬다.

> 이벤트를 사용하면 코드가 다소 복잡해지는 대신 도메인 간의 의존성이나 외부 시스템에 대한 의존을 낮춰주는 장점을 얻을 수 있다. 또한 시스템을 확장하는 데에 이벤트가 핵심 역할을 수행하게 된다.

예제 도메인
```java
public class Member {
    private Password password;
    
    public void initializePassword() {
        String newPassword = generateRandomPassword();
        this.password = new Password(newPassword);
        Events.raise(new PasswordChangedEvent(this.id, password));
    }
}
```

예제 서비스
```java
public class InitPasswordService {
    
    @Transactional
    public void initializePassword(string memberId) {
        Events.handle((PasswordChangedEvent evt) -> {
            // evt.getId()에 해당하는 회원에게 이메일을 발송하는 기능 구현
        });
        
        Member member = memberRepository.findById(memberId);
        checkMemberExists(member);
        member.initializePassword();
    }
}
```

### 값 검증

값 검증은 표현 영역과 응용 서비스 두 곳에서 모두 수행할 수 있다.

예시로 
```java
@Controller
public class Controller {
    @RequestMapping
    public String join(JoinRequest joinRequest, Errors errors) {
        try {
            joinService.join(joinRequest);
            return successView;
        } catch(EmptyPropertyException ex) {
            errors.rejectValue(ex.getPropertyName(), "empty");
            return formView;
        } catch(InvalidPropertyException ex) {
            errors.rejectValue(ex.getPropertyName(), "invalid");
            return formView;
        } catch(DuplicateIdexception ex) {
            errors.rejectValue(ex.getPropertyName(), "duplicate");
            return formView;
        }
    }
}
```
위 코드의 문제점은 응용 서비스에서 값을 검사하는 시점에 첫 번째 값이 올바르지 않아 익셉션을 발생시키면 나머지 항목에 대해서는 값을 검사하지 않게 된다. 이러면 사용자는 첫 번째 값에 대한 에러 메시지만 보게 되고 나머지 항목에 대해서는 값이 올바른지 여부를 알 수 없게 된다. 이는 사용자가 같은 폼에 값을 여러 번 입력하게 만든다.

> 이럴 땐 응용 서비스에 값을 전달하기 전에 표현 영역에서 값을 검사하면 된다.

```java
@Controller
public class Controller {
    @RequestMapping
    public String join(JoinRequest joinRequest, Errors, errors) {
        checkEmpty(joinRequest.getId(), "id", errors);
        checkEmpty(joinRequest.getName(), "name", errors);
        ...
        
        
        // 모든 값의 형식을 검증한 뒤, 에러가 존재하면 다시 폼을 보여줌
        if (errors.hasErrors()) return formView;
        
        try {
            joinService.join(joinRequest);
            return successView;
        } catch (DuplicationIdException ex) {
            errors.rejectValue(ex.getPropertyName(), "duplicate");
            return formView;
        }
    }
    
    private void checkEmpty(String value, String property, Errors errors) {
        if (isEmpty(vlaue)) errors.rejectValue(property, "empty");
    }
}
```

스프링은 Validator 인터페이스를 별도로 제공해서 코드를 간결하게 줄일 수 있다.

```java
@Controller
public class Controller {
    @RequestMapping
    public String join(JoinRequest joinRequest, Errors errors) {
        new JoinRequestValidator().validate(joinRequest, errors);
        if (errors.hasErrors()) return formView;
        
        ...
    }
}
```

이렇게 표현 영역에서 필수 값과 값의 형식을 검사하면 실질적으로 `응용 서비스는 아이디 중복 여부와 같은 논리적 오류만 검사하면 된다`.

- 표현 영역 : 필수 값, 값의 형식, 범위 등을 검증
- 응용 서비스 : 데이터의 존재 유무와 같은 논리적 오류를 검증한다.

> 엄격하게 하려면 표현, 응용 영역 둘다에 검증 과정을 넣으면 되지만 복잡해지잖아? 근데 응용 서비스를 실행하는 주체가 다양하면 응용 서비스에서 반드시 파라미터로 전달받은 값이 올바른지 검사해야함!!
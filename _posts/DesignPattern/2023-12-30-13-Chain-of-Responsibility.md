---
title: "[DesignPattern] 책임 연쇄 패턴 (Chain Of Responsibility)"

layout: post
categories: DesignPattern

toc: true
toc_sticky: DesignPattern

date: 2023-12-30
last_modified_at: 2023-12-30
---

# 책임 연쇄 패턴 (Chain Of Responsibility)

메시지를 보내는 객체와 이를 받아 처리하는 객체들 간의 결합도를 없애기 위한 패턴입니다. 하나의 요청에 대한 처리가 반드시 한 객체에서만 되지 않고, 여러 객체에게 그 처리 기회를 주려는 것입니다.


## 동기

이 패턴은 메시지 송신 측과 수신측을 분리한는 것이다. 이를 위해 이 요청을 처리하는 기회를 다른 객체에 분산한다. 요청은 실제 이 요청을 처리할 객체를 찾을 때까지 객체 연결고리를 따라 전달된다.

즉, 송신하는 측이 자신이 아는 주체에게 처리를 요청하면, 이를 수신한 객체가 자신과 연결된 고리를 따라서 계속 이 요청을 전달하고, 이 중에 어느 한 객체가 실제 상황에 적합하다고 판단되면 자신에게 정의된 서비스를 제공한다.


## 활용성

책임 연쇄 패턴은 다음의 경우에 사용 할 수 있다.

- 하나 이상의 객체가 요청을 처리해야 하고, 그 요청 처리자 중 어떤 것이 선행자 인지 모를 때, 처리자가 자동으로 확정되어야 한다.
- 메시지를 받을 객체를 명시하지 않은 채 여러 객체 중 하나에게 처리를 요청하고 싶을 때
- 요청을 처리할 수 있는 객체 집합이 동적으로 정의되어야 할 때


## 구조

![책임 연쇄 패턴 구조1]({{site.url}}/public/image/2023/2023-12/COR001.png)

전형적인 객체 구조는 다음처럼 보일 수 있다.

![책임 연쇄 패턴 구조2]({{site.url}}/public/image/2023/2023-12/COR002.png)

- Handler: 요청을 처리하는 인터페이스를 정의하고, 후속 처리자(successor)와 연결을 구현합니다. 즉, 연결 고리에 다음 객체에게 다시 메시지를 보냅니다.
- ConcreteHandler: 책임져야 할 행동이 있다면 스스로 요청을 처리하여 후속 처리자에 접근할 수 있습니다. 즉, 자신이 처리할 행동이 있으면 처리하고, 그렇지 않으면 후속 처리자에게 다시 처리를 요청합니다.
- Client: `ConcreteHandler` 객체에게 필요한 요청을 보냅니다.

## 결과

책임 연쇄 패턴은 다음과 같은 장점과 단점이 있습니다.

1. **객체 간의 행동적 결합도가 적어집니다.** 다른 객체가 어떻게 요청을 처리하는지 몰라도 됩니다. 단지 요청을 보내는 객체는 이 메시지가 적절하게 처리될 것이라는 것만 확인하면 된다. 결과적으로 이 패턴은 객체들 간의 상호작용을 단순화 시킨다. 객체가 관련된 모든 후보를 다 알필요는 없이 자신과 연결된 단 하나의 후보만 알면 된다.
2. **객체에게 책임을 할당하는 데 유연성을 높일 수 있다.** 객체의 책음을 여러객체에게 분산시킬 수 있으므로 런타임에 객체 연결고리를 변경하거나 추가하여 책임을 변경하거나 확장할 수 있다.
3. **메시지 수신이 보장되지는 않습니다.** 어떤 객체가 이 처리에 대한 수신을 담당한다는 것을 명시하지 않으므로 요청이 처리된다는 보장은 없다. 만약 객체들 간의 연결고리가 잘 정의되어있지 않다면, 요청은 처리되지 못한 채로 버려질 수 있다.


## 구현

### 적용전

Client 에서 메시지를 보내 출력하는 로직을 처리하고자 한다. Client에서는 동작시키고자 하는 요청을 Handler에게 보내 출력하도록 처리 할 수 있다.

**Client**

```java
public class Client {
    public static void main(String[] args) {
        Request request = new Request("이 요청을 출력해 주세요.");
        requestHandler.handler(request);
    }
}
```

`RequestHandler`는 요청을 받아 문자를 출력하는 역할을 처리하는 Handler이다.

**RequestHandler**

```java
public class RequestHandler {
    public void handler(Request request) {
        System.out.println(request.getBody());
    }
}
```

만약 여기에 인증된 사용자만 문자를 출력할 수 있다는 기능을 추가하려면 어떻게 해야할까? 기존의 `RequestHandler`의 handler를 수정하여 기능을 추가로 붙이는 것은 OOP의 법칙중 단일 책임 원칙(Single Responsibility Principle)에 어긋날 수 있다.

인증 역할을 하는 handler인 `AuthRequestHandler`를 추가해 보도록하자.

**AuthRequestHandler**

```java
public class AuthRequestHandler extends RequestHandler {
    public void handler(Request request) {
        System.out.println("인증을 시작합니다.");
        System.out.println("인증을 종료되었습니다.");
        super.handler(request);
    }
}

```

이제 인증을 진행하기 위해 `AuthRequestHandler`를 추가한다면 Client는 추가적인 기능인 인증 기능을 가진 handler의 정보를 알고 있어야한다.

**Client**

```java
public class Client {
    public static void main(String[] args) {
        Request request = new Request("이 요청을 출력하여 처리해주세요.");
        RequestHandler requestHandler = new AuthRequestHandler();
        requestHandler.handler(request);
    }
}
```

만약 로그를 출력하는 것과 같이 또다른 기능이 추가된다면 Client는 `AuthRequestHandler` 뿐만 아니라, `LogRequestHandler`등 확장된 기능을 가진 handler여부를 모두 알고 있어야 한다.

또한 인증과, 로그의 출력을 동시에 진행하기 위해서는 Client에서 어떻게 사용해야 하는지 문제가 될것이다.

정리하자면 책임 연쇄 패턴을 적용하는 목적은 클라이언트가 구체적인 Handler타입을 모르게 한뒤 원하는 기능들의 작업을 연쇄적으로 처리할 수 있게 만드는 것이 목적이라고 할 수 있다.


### 적용후

연쇄적으로 ReuqstHandler가 이어질 수 있게 끔 추상 클래스를 작성해서 연결된 Handler가 있으면 해당 Handler를 실행 시킬 수 있도록 클래스를 구현합니다.

`nextHandler`가 null일 경우 chain이 종료되도록 로직을 작성한다.

**RequestHandler**

```java
public abstract class RequestHandler {

    private RequestHandler nextHandler;

    RequestHandler(RequestHandler nextHandler) {
        this.nextHandler = nextHandler;
    }

    public void handler(Request request) {
        if (nextHandler != null)
            nextHandler.handler(request);
    }
}
```

그 다음 추상 클래스 `RequestHandler`를 상속 받아 인증 진행(AuthRequestHandler), 로그 출력(LoggingRequestHandler), 문구 출력(PrintRequestHandler)에 해당 하는 Handler들을 작성해준다.

**AuthRequestHandler**

```java
public class AuthRequestHandler extends RequestHandler{
    public AuthRequestHandler(RequestHandler nexthandler) {
        super(nexthandler);
    }

    @Override
    public void handler(Request request) {
        System.out.println("인증을 시작합니다.");
        System.out.println("인증을 종료되었습니다.");
        super.handler(request);
    }
}
```

**LoggingRequestHandler**

```java
public class LoggingRequestHandler extends RequestHandler{
    public LoggingRequestHandler(RequestHandler nexthandler) {
        super(nexthandler);
    }

    @Override
    public void handler(Request request) {
        System.out.println("handelr 로그를 출력합니다.");
        super.handler(request);
    }
}
```

**PrintRequestHandler**

```java
public class PrintRequestHandler extends RequestHandler{
    public PrintRequestHandler(RequestHandler nexthandler) {
        super(nexthandler);
    }

    @Override
    public void handler(Request request) {
        System.out.println(request.getBody());
        super.handler(request);
    }
}
```

Client에서는 RequestHandler들을 조립하는 chain을 조립하여 handler를 실행하면 된다.

**Client**

```java
public class Client {

    private RequestHandler requestHandler;

    public Client(RequestHandler requestHandler) {
        this.requestHandler = requestHandler;
    }

    public void doWork() {
        Request request = new Request("이 문구를 출력해주세요.");
        requestHandler.handler(request);
    }

    public static void main(String[] args) {
        RequestHandler chain = new AuthRequestHandler(new LoggingRequestHandler(new PrintRequestHandler(null)));
        Client client = new Client(chain);
        client.doWork();
    }
}
```

Client는 구체적인 핸들러 타입으로부터 요청을 처리하는 쪽과 요청을 보내는 쪽의 연결의 결합도가 느슨해지게 된다.

또한 Client 코드가 더이상 구체적인 어떤 핸들러를 써서 요청을 처리할지 몰라도 되고, 이 체인을 만들어 주는 곳에서 체인이 결정되게 된다.




---
title:  "[Java] Spring boot에서 graceful shutdown을 적용하자"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2023-04-28
last_modified_at: 2022-04-28
---

최근 서버에 배포시마다 연결이 끊김에 따라, 해당 해결방법을 찾던 도중 Spring boot에서 graceful shutdown 이란 기능을 지원하는 것을 알게 되었다.

graceful [우아한] 이란 뜻이며, 말그대로 서버를 종료할때 갑자기 종료 시켜버리는 것이 아닌, 마저 하던 일을 종료하고 우아하게 종료하는 것을 말한다.

반대는 hard shutdown으로, 현재 실행되는 일은 상관없이 강제로 종료하는 것을 말한다.

버전마다 설정하는 방법은 Spring Boot 2.3 버전 전후로 차이가 나뉘어 진다.
먼저 Spring Boot 2.3 이상일 경우부터 알아보도록 하자.


## Spring Boot 2.3 이상의 설정

Spring Boot에서 graceful shutdown 설정을 적용하는 것은 간단하다. yml에 다음과 같은 설정만 추가해 주면 된다.

```yaml
server:
  shutdown: graceful
```

그럼 해당 설정을 해주고 적용 유무를 확인해 보도록 하자.

![graceful01]({{site.url}}/public/image/2023/2023-04/28-graceful001.png)

서버를 기동시킨후 Stop 버튼을 눌러 서버를 종료해 보자.

그러면 아래의 Console에 graceful shutdown을 통해 서버가 Stop되었다는 로그가 출력된다.

```shell
2023-04-28T22:45:12.152+09:00  INFO 28976 --- [ionShutdownHook] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete
2023-04-28T22:45:12.171+09:00  INFO 28976 --- [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete
```


## 테스트를 해보자.

그렇다면 Request가 들어와 수행중인 상태에서 서버를 Stop시킨다면 어떻게 될까? 간단한 테스트를 통해 알아보도록 하자.

아래의 예제코드는 `http://localhost:8080/api/v1/graceful` 을 호출할 시 `gracefulTest in` 를 출력하고, 10초 후 다시 `gracefulTest out`를 출력하는 코드이다.

```java
@Slf4j
@RestController
@RequestMapping("/api/v1")
public class GracefulController {

    @GetMapping("/graceful")
    public void gracefulTest() throws InterruptedException {
        log.info("gracefulTest in");
        Thread.sleep(10000);
        log.info("gracefulTest out");
    }
}
```

만약 해당 로직이 실행중일때 Stop을 눌러 서버를 종료한다면 어떻게 될까?

아래와 같이 `gracefulTest in`을 통해 request가 해당 API에 들어온것을 확인 할 수 있었으며, `Waiting for active requests to complete`의 문구로 인해 request가 종료될때 까지 shutdown을 기다린다는 것을 알 수 있다.
이후 `gracefulTest out`이 출력된 후 서버가 종료된것을 볼 수있다.

```shell
2023-04-28T23:14:20.497+09:00  INFO 13340 --- [nio-8080-exec-2] c.e.d.g.controller.GracefulController    : gracefulTest in
2023-04-28T23:14:24.037+09:00  INFO 13340 --- [ionShutdownHook] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete
2023-04-28T23:14:30.508+09:00  INFO 13340 --- [nio-8080-exec-2] c.e.d.g.controller.GracefulController    : gracefulTest out
2023-04-28T23:14:30.538+09:00  INFO 13340 --- [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete
```

만약 여기서 graceful shutdown이 request를 종료하는동안 새로운 request가 들어온다면 어떻게 될까?

Stop을 누르고 새로운 요청을 넣자 추가적인 요청이 들어가지 않는것을 볼 수 있다.

![graceful02]({{site.url}}/public/image/2023/2023-04/28-graceful002.png)

여기서 한가지 주의할 점이 생긴다. 만약 모종의 이유로 해당 API에 Deadlock이 발생한다면 어떻게 될까?

graceful shutdown은 교착상태에 빠진 Request가 종료되길 계속 기다리기 때문에 결국 종료되지 않고 무한 대기하게 된다.

그래서 timeout을 설정해 shutdown을 할때 진행중인 request에 대해 몇초를 기다릴지 설정 해 줄 수 있다.

설정은 아래와 같이 해주면 된다. 10초를 기다린 후 연결을 끊고 shutdown을 진행한다는 설정이다.

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 10s
```

부득이하게 API마다 오래 걸리는 API가 있을 수 있으니 적절한 timeout설정이 필요하다.


## Spring Boot 2.3 미만의 설정

그렇다면 Spring Boot 2.3 미만 버전에서는 어떻게 설정을 해줘야 할까?

Spring에서 따로 제공하는 것이 없기 때문에 직접 구현을 해주어야 한다.

먼저 어떤 서버를 사용하느냐에 따라 상속받는 구현체가 달라진다. 해당 예시를 사용할 때에는 Spring의 내장된 기본 Tomcat을 사용했으므로 Tomcat 기준으로 설명하고자 한다.

먼저 `TomcatConnectorCustomizer` 인터페이스의 구현체를 하나 만들어 준다.


### GracefulShutdownTomcatConnector

```java
@Component
public class GracefulShutdownTomcatConnector implements TomcatConnectorCustomizer {

    private volatile Connector connector;

    @Override
    public void customize(Connector connector) {
        this.connector = connector;
    }

    public Connector getConnector() {
        return connector;
    }
}
```

`TomcatConnectorCustomizer`를 구현하게 되면 Spring에 내장된 톰캣 Connector의 설정을 변경 할 수 있게 된다.

`volatile` 키워드는 해당 키워드가 선언된 변수를 읽어 들일때 CPU 캐시가 아닌 메인 메모리에서 읽어 오도록 하는 키워드이다.

이후 `TomcatConnectorCustomizer`를 구현한 `GracefulShutdownTomcatConnector` Class를 Tomcat에 등록을 해주도록 하자.


### TomcatConfig

```java
@Configuration
public class TomcatConfig {

    private final GracefulShutdownTomcatConnector gracefulShutdownTomcatConnector;

    public TomcatConfig(GracefulShutdownTomcatConnector gracefulShutdownTomcatConnector) {
        this.gracefulShutdownTomcatConnector = gracefulShutdownTomcatConnector;
    }

    @Bean
    public ConfigurableServletWebServerFactory webServerFactory() {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.addConnectorCustomizers(gracefulShutdownTomcatConnector);

        return factory;
    }
}
```

`TomcatConfig` 클래스의 `webServerFactory` 메소드에서 `TomcatServletWebServerFactory`의 Factor 객체를 이용하여 위에서 구현한 `GracefulShutdownTomcatConnector` Class를 등록하는 것을 확인 할 수 있다.

이제 Tomcat에 대한 설정은 마무리가 되었다.

Application 서버가 종료될 경우 Eventlistener를 구현해 graceful shutdown을 구현 해주도록 하자.


### GracefulShutdownListener

```java
@Slf4j
@Component
public class GracefulShutdownListener implements ApplicationListener<ContextClosedEvent> {

    private static final int GRACEFUL_SHUTDOWN = 30;

    private final GracefulShutdownTomcatConnector gracefulShutdownTomcatConnector;

    public GracefulShutdownListener(GracefulShutdownTomcatConnector gracefulShutdownTomcatConnector) {
        this.gracefulShutdownTomcatConnector = gracefulShutdownTomcatConnector;
    }

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        this.gracefulShutdownTomcatConnector.getConnector().pause();

        ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) this.gracefulShutdownTomcatConnector.getConnector()
                .getProtocolHandler()
                .getExecutor();

        log.info("Commencing graceful shutdown. Waiting for active requests to complete");
        threadPoolExecutor.shutdown();
        try {
            threadPoolExecutor.awaitTermination(this.GRACEFUL_SHUTDOWN, TimeUnit.SECONDS);

            log.info("Graceful shutdown complete");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();

            log.error("Graceful Shutdown Failed");
        }
    }
}
```

먼저 `ApplicationListener<ContextClosedEvent>` 인터페이스는 Application 서버에 대한 종료가 감지될때 한번 실행되는 인터페이스 구현 메서드 이다.

이 인터페이스를 구현하여, Request가 종료되지 않았을 경우 기다리는 로직을 구현할 수 있다.

종료 이벤트가 실행되는 동안 `GRACEFUL_SHUTDOWN` 변수에 설정한 시간 만큼 대기하게 되며, 해당 시간이 지날경우 application 서버를 shutdown 한다.


## 실행

다시 한번 위의 동일한 테스트코드를 통해 Application 서버의 종료를 테스트 해보도록 하자.

`GracefulShutdownListener` 구현 클래스에서 동일한 결과의 logging을 확인할 수 있다.

```shell
2023-04-29T01:01:09.811+09:00  INFO 33244 --- [nio-8080-exec-5] c.e.d.g.controller.GracefulController    : gracefulTest in
2023-04-29T01:01:11.312+09:00  INFO 33244 --- [ionShutdownHook] c.e.d.g.l.GracefulShutdownListener       : Commencing graceful shutdown. Waiting for active requests to complete
2023-04-29T01:01:29.814+09:00  INFO 33244 --- [nio-8080-exec-5] c.e.d.g.controller.GracefulController    : gracefulTest out
2023-04-29T01:01:29.847+09:00  INFO 33244 --- [ionShutdownHook] c.e.d.g.l.GracefulShutdownListener       : Graceful shutdown complete
```

### 참고 사이트

https://blog.naver.com/PostView.nhn?blogId=debugrammer&logNo=221710570569



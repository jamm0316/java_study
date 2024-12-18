[<back](https://www.notion.so/Spring-basic-41cfc2c136874d9896aa115923bd4109?pvs=21)

---

<aside>
📃

Contents

</aside>

---

## 빈 생명주기 콜백 시작

데이터베이스 커넥션 풀이나, 네트워크 소켓처럼 애플리케이션 시작 시점에 필요한 연결을 미리 해두고, 애플리케이션 종료 시점에 연결을 모두 종료하는 작업을 진행하려면, 객체의 초기화와 종료 작업이 필요하다.
이번 시간에는 스프링을 통해 이러한 초기화 작업과 종료 작업을 어떻게 진행하는지 예제로 알아보자.

간단하게 외부 네트워크에 미리 연결하는 객체를 하나 생성한다고 가정하자. 실제로 네트워크에 연결하는 것이 아니라, 단순히 문자로만 출력되도록 했다. 이 `NetworkClient`는 애플리케이션 시작 시점에 `connect()`를 호출해서 연결을 맺어두어야 하고, 애플리케이션이 종료되면 `disConnect()`를 호출해서 연결을 끊어야 한다.

**예제 코드**

`core/src/test/java/hello/core/lifecycle/NetworkClient.java`

**NetworkClient**

```java
package hello.core.lifecycle;

public class NetworkClient {

    private String url;

    public NetworkClient() {
        System.out.println("생성자 호출, url = " + url);
        connect();
        call("초기화 연결 메세지");
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public void connect() {
        System.out.println("connect: " + url);
    }

    public void call(String message) {
        System.out.println("call: " + url + " message: " + message);
    }

    public void disconnect() {
        System.out.println("close: " + url);
    }
}
```

`core/src/test/java/hello/core/lifecycle/BeanLifeCycleTest.java`

**BeanLifeCycleTest**

```java
public class BeanLifeCycleTest {

    @Test
    void lifeCycleTest() {
        ConfigurableApplicationContext ac = new AnnotationConfigApplicationContext(LifeCycleConfig.class);
        NetworkClient client = ac.getBean(NetworkClient.class);
        ac.close();
    }

    @Configuration
    static class LifeCycleConfig {

        @Bean
        public NetworkClient networkClient() {
            NetworkClient networkClient = new NetworkClient();
            networkClient.setUrl("http://hello-spring.dev");
            return networkClient;
        }
    }
}
```

**실행결과**

```java
생성자 호출, url = null
connect: null
call: null message: 초기화 연결 메세지
```

생성자 부분을 보면 url 정보 없이 connect가 호출되는 것을 확인할 수 있다.

너무 당연한 이야기지만, 객체를 생성하는 단계에는 url이 없고, 객체를 생성한 다음에 외부에서 수정자 주입을 통해서 `setUrl()`이 호출되어야 url이 존재하게 된다.

(다시말해, **NetworkClient**의 생성자 생성 시점에 아래와 같은 코드가 실행되고

```java
public NetworkClient() {
    System.out.println("생성자 호출, url = " + url);
    connect();
    call("초기화 연결 메세지");
}
```

이후 수정자 주입(`setUrl`)을 통해 `url`이 등록되므로 최종적으로 반환되는 `networkClient`에는 `“http://hello-spring.dev”` 라는 `url`이 등록되어있는 객체가 반환되고, 출력값은 `null`값이 반환되는 것이다.

스프링 빈은 간단하게 다음과 같은 라이프사이클을 가진다.

**객체 생성 → 의존관계 주입**

스프링 빈은 객체를 생성하고, 의존관계 주입이 다 끝난다음에야 필요한 데이터를 사용할 수 있는 준비가된다. 따라서 초기화 작업은 의존관계 주입이 모두 완료되고 난 다음에 호출해야한다. 그런데 개발자가 의존과계 주입이 모두 완료된 시점을 어떻게 알 수 있을까?

스프링은 의존관계 주입이 완료되면 스프링 빈에게 콜백 메서드를 통해서 초기화 시점을 알려주는 다양한 기능을 제공한다. 또한 스프링은 스프링 컨테이너가 종료되기 직전에 소멸 콜백을 준다. 따라서 안전하게 종료 작업을 진행할 수 있다.

**스프링 빈의 이벤트 라이프사이클**

**스프링 컨테이너 생성 → 스프링 빈 생성 → 의존관계 주입 → 초기화 콜백 → 사용 → 소멸전 콜백 → 스프링 종료**

- 초기화 콜백: 빈이 생성되고, 빈의 의존관계 주입이 완료된 후 호출
- 소멸전 콜백: 빈이 소멸되기 직전 호출

스프링은 다양한 방식으로 생명주기 콜백을 지원한다.

> **참고: 객체 생성과 초기화를 분리하자.**
생성자는 필수 정보(파라미터)를 받고, 메모리 할당해서 객체를 생성하는 책임을 가진다. 반면에 초기화는 이렇게 생성된 값들을 활용해서 외부 커넥션을 연결하는등 무거운 동작을 수행한다.
따라서 생성자 안에서 무거운 초기화 작업을 함께 하는 것 보다는 객체를 생성하는 부분과 초기화 하는 부분을 명확하게 나누는 것이 유지보수 관점에서 좋다. 물론 초기화 작업이 내부 값들만 약간 변경하는 정도로 단순한 경우에는 생성자에서 한번에 다 처리하는 게 더 나을 수 있다.
> 

**스프링은 크게 3가지 방법으로 빈 생명주기 콜백을 지원한다.**

- 인터페이스(InitializingBean, DisposableBean)
- 설정 정보에 초기화 메서드, 종료 메서드 지정
- `@PostConstruct`, `@PreDestory` 애노테이션 지원

## 인터페이스 InitializingBean, DisposableBean

**NetworkClient**

```java
public class NetworkClient implements InitializingBean, DisposableBean {

    private String url;

    public NetworkClient() {
        System.out.println("생성자 호출, url = " + url);
    }

    ...

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("NetworkClient.afterPropertiesSet");
        connect();
        call("초기화 연결 메세지");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("NetworkClient.destroy");
        disconnect();
    }
}
```

**BeanLifeCycleTest**

```java
public class BeanLifeCycleTest {

    @Test
    void lifeCycleTest() {
        ConfigurableApplicationContext ac = new AnnotationConfigApplicationContext(LifeCycleConfig.class);
        NetworkClient client = ac.getBean(NetworkClient.class);
        ac.close();
    }

    @Configuration
    static class LifeCycleConfig {

        @Bean
        public NetworkClient networkClient() {
            NetworkClient networkClient = new NetworkClient();
            networkClient.setUrl("http://hello-spring.dev");
            return networkClient;
        }
    }
}
```

- `InitializingBean` 은 `afterPropertiesSet()` 메서드로 초기화를 지원한다.
- `DisposableBean`은 `destroy`() 메서드로 소멸을 지원한다.
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6b8d40ba-5287-42be-84df-56b1c96a2c05/815684f3-12a2-44aa-bdaa-58bd894afc4c/image.png)
    

**실행결과**

```java
생성자 호출, url = null  -> 생성자 호출 시점
NetworkClient.afterPropertiesSet  -> 의존관계 주입 시점
connect: http://hello-spring.dev
call: http://hello-spring.dev message: 초기화 연결 메세지
NetworkClient.destroy  -> 스프링 컨테이너 종료 시점
close: http://hello-spring.dev
```

- 출력 결과를 보면 초기화 메서드가 주입 완료 후에 적절하게 호출된 것을 확인할 수 있다.
- 그리고 스프링 컨테이너의 종료가 호출되자 소멸 메서드가 호출 된 것도 확인할 수 있다.

**초기화, 소멸 인터페이스 단점**

- 이 인터페이스는 스프링 전용 인터페이스다. 해당 코드가 스프링 전용 인터페이스에 의존한다.
- 초기화, 소멸 메서드 이름을 변경할 수 없다.
- 내가 코드를 고칠 수 없는 외부라이브러리에 적용할 수 없다.

> 참고: 인터페이스 사용하는 초기화, 종료 방법은 스프링 초창기에 나온 방법들이고, 지금은 다음의 더 나은 방법들이 있어서 거의 사용하지 않는다.
> 

## 빈 등록 초기화, 소멸 메서드

설정 정보에 `@Bean(initMethod = “init”, destroyMethod = “close”)` 처럼 초기화, 소멸 메서드를 지정할 수 있다.

**설정 정보를 사용하도록 변경**

**NetworkClient**

```java
public class NetworkClient {

    private String url;

    public NetworkClient() {
        System.out.println("생성자 호출, url = " + url);
    }

   ...

    public void init() throws Exception {
        System.out.println("NetworkClient.init");
        connect();
        call("초기화 연결 메세지");
    }

    public void close() throws Exception {
        System.out.println("NetworkClient.close");
        disconnect();
    }
}
```

**BeanLifeCycleTest**

```java
public class BeanLifeCycleTest {

    @Test
    void lifeCycleTest() {
        ConfigurableApplicationContext ac = new AnnotationConfigApplicationContext(LifeCycleConfig.class);
        NetworkClient client = ac.getBean(NetworkClient.class);
        ac.close();
    }

    @Configuration
    static class LifeCycleConfig {

        @Bean(initMethod = "init", destroyMethod = "close")
        public NetworkClient networkClient() {
            NetworkClient networkClient = new NetworkClient();
            networkClient.setUrl("http://hello-spring.dev");
            return networkClient;
        }
    }
}
```

**실행결과**

```java
생성자 호출, url = null
NetworkClient.init
connect: http://hello-spring.dev
call: http://hello-spring.dev message: 초기화 연결 메세지
NetworkClient.close
close: http://hello-spring.dev
```

**설정 정보 사용 장점**

- 메서드 이름을 자유롭게 줄  수 있다.
- 스프링 빈이 스프링 코드에 의존하지 않는다.
- 코드가 아니라 설정 정보를 사용하기 때문에 코드를 고칠 수 없는 외부 라이브러리에 초기화, 종료 메서드를 적용할 수 있다.

**종료 메서드 추론**

- `@Bean`의 `destroyMethod` 속성에는 아주 특별한 기능이 있다.
- 라이브러리는 대부분 `close`, `shutdown`이라는 이름의 종료 메서드를 사용한다.
- `@Bean`의 `destroyMethod`는 기본값이 `(inferred)`(추론)으로 등록되어 있다.
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6b8d40ba-5287-42be-84df-56b1c96a2c05/1bfe35de-59c5-4c7f-91bd-af7d086ab9e6/image.png)
    
- 이 추론기능은 `close`, `shutdown`라는 이름의 메서드를 자동으로 호출해준다. 이름 그대로 종료 메서드를 추론해서 호출 해준다.
- 따라서 직접 스프링 빈으로 등록하면 종료 메서드는 따로 적어주지 않아도 잘 동작한다.
- 추론 기능을 사용하기 싫으면 `destoryMethod=””` 처럼 빈 공백을 지정한면 된다.

## 애노테이션 @PostConsturct, @PreDestory

**NetworkClient**

```java
public class NetworkClient {

    private String url;

    public NetworkClient() {
        System.out.println("생성자 호출, url = " + url);
    }
...
    @PostConstruct
    public void init() throws Exception {
        System.out.println("NetworkClient.init");
        connect();
        call("초기화 연결 메세지");
    }

    @PreDestroy
    public void close() throws Exception {
        System.out.println("NetworkClient.close");
        disconnect();
    }
}
```

**BeanLifeCycleTest**

```java
public class BeanLifeCycleTest {

    @Test
    void lifeCycleTest() {
        ConfigurableApplicationContext ac = new AnnotationConfigApplicationContext(LifeCycleConfig.class);
        NetworkClient client = ac.getBean(NetworkClient.class);
        ac.close();
    }

    @Configuration
    static class LifeCycleConfig {

        @Bean
        public NetworkClient networkClient() {
            NetworkClient networkClient = new NetworkClient();
            networkClient.setUrl("http://hello-spring.dev");
            return networkClient;
        }
    }
}
```

**실행결과**

```java
생성자 호출, url = null
NetworkClient.init
connect: http://hello-spring.dev
call: http://hello-spring.dev message: 초기화 연결 메세지
NetworkClient.close
close: http://hello-spring.dev
```

**@PostConstruct, @PreDestory 애노테이션 특징**

- 최신 스프링에서 가장 권장하는 방법.
- 애노테이션 하나만 붙이면 되므로 매우 편리하다.
- 패키지를 잘 보면 `import jakarta.annotation.PostConstruct;` 이다. 스프링에 종속적인 기술이 아니라 JSR-250라는 자바 표준 이다. 따라서 스프링이 아닌 다른 컨테이너에서도 동작한다.
- 컴포넌트 스캔과 잘 어울린다.
- 유일한 단점은 외부 라이브러리에서 동작하지 않는다는 것. 외부 라이브러리를 초기화, 종료 해야 하면 @Bean의 기능을 사용하자.

**정리**

- **@PostConstruct, @PreDestory 애노테이션을 사용하자.**
- 코드를 고칠 수 없는 외부 라이브러리를 초기화, 종료해야 하면 @Bean의 `initMethod`, `destroyMethod`를 사용하자.
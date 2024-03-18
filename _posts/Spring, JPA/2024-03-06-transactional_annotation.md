---
layout: post
title: 동작원리를 통해 알아보는 @Transactional 의 올바른 사용
category: Spring
excerpt: 트랜잭션 기능을 추상화시켜 놓으면 트랜잭션을 동작시키는 기술이 바뀌었을 때 모든 코드를 변경할 필요 없이 쉽게 변경할 수 있다. DataSource 를 통해 커넥션을 사용하는 기능은 인터페이스가 아닌, 각 구현체에 있다. jdbc 나 hibernate 같은 의존성을 포함시키면 DataSourceTransactionManager, HibernateTransactionManager 같은 구현체가 PlatformTransactionManager 를 상속받아 구현된다. 이후 @EnableTransactionManagement 를 통해 선언적 트랜잭션 관리를 활성화하고 프록시를 생성하는 방식을 선택한다. (선언적 트랜잭션 관리란, @Transactional 를 통해 트랜잭션을 관리하는 것을 말한다)

---

트랜잭션은 DB 와 관련된 작업의 단위를 말한다. 트랜잭션은 최소한의 단위로써 더 이상 쪼갤 수 없다. 

트랜잭션은 다양한 상황에서 사용된다. 항공편 예약을 한다고 해보자. 항공편 예약은 다음 두 가지 동작이 합쳐져 만들어진다.

- 고객 정보에 항공편 좌석 예약
- 좌석 수 줄이기(예약된 좌석 구매 불가)

위 두 가지 동작이 같이 이루어져야 한다. 만약 둘 중 하나만 성공하면 큰 문제가 생긴다. 
예를 들어 예약을 했는데 좌석수가 줄어들지 않으면 중복예약이 될 수도 있다.

위와 같은 상황에서 트랜잭션을 사용한다. 트랜잭션의 역할은 간단하다. 
- 동작이 진행되다가 중간에 문제가 생기면 rollback 
- 모든 동작이 잘 진행되면 DB 에 commit

# @Transactional 
스프링에서는 트랜잭션 관리를 위해 `@Transactional` 을 제공한다. 

과거엔 JDBC 트랜잭션 코드를 아래와 같이 짰다.
{% highlight Java%}
String URL = "jdbc:your_database_url";
String USERNAME = "username";
String PASSWORD = "password";

Connection conn = null;
try {
    conn = DriverManager.getConnection(URL, USERNAME, PASSWORD);
    conn.setAutoCommit(false); // 트랜잭션 시작

    //항공편 예약 SQL 로직
   
    conn.commit(); // 트랜잭션 커밋
} catch(SQLException e) {
    conn.rollback(); //예외 생기면 롤백
    throw e;
} finally {
    conn.close(); //커넥션 종료
}
{% endhighlight %}
try ~catch 문으로 처리하는 방식이다. 하지만 이 방법엔 문제가 많다. 
- 트랜잭션을 유지하기 위해선 같은 커넥션을 사용해야 한다. 그러기 위해선 파라미터로 동일한 커넥션을 계속 넘겨야 한다. 이는 유지보수하기 어려운 코드를 만든다.
- `SQLException` 은 JDBC 에 속한 예외다. 따라서 Repository 단에서 처리하는게 맞다.
- 커넥션을 직접 연결하고 해제하는 코드와, try ~catch 문의 코드가 계속해서 반복된다.
- 서비스 계층에서 트랜잭션을 처리하는 건 맞지만, 특정 기술(SQLException - JDBC)에 의존하면 안된다.

위와 같은 문제점 때문에 `@Transactional` 이 등장했다. 
{% highlight Java%}
@Transactional
public void reserveFlight(Long memberId, Long flightId) {
    //항공편 예약 로직
}
{% endhighlight %}
`@Transactional` 하나만 붙임으로써 위의 문제가 상당부분 해결되었다(참고로 예외처리는 별개임)

복잡한 트랜잭션 처리를 `@Transactional` 이 알아서 해준 것이다. 구체적으로 어떻게 동작할까?
## @Transactional 동작원리
`@Transactional` 은 AOP 기반으로 proxy 객체를 활용하여 동작한다.

이를 이해하기 위해선 먼저 `TransactionAutoConfiguration` 클래스에 대해 알아야 한다.

이 클래스는 트랜잭션을 사용할 경우 해당 설정정보를 포함하고 있는 설정파일이다. 

[TransactionAutoConfiguration.java](https://github.com/spring-projects/spring-boot/blob/v3.2.3/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/transaction/TransactionAutoConfiguration.java)
{% highlight Java%}
@AutoConfiguration
@ConditionalOnClass(PlatformTransactionManager.class)
public class TransactionAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnSingleCandidate(ReactiveTransactionManager.class)
    public TransactionalOperator transactionalOperator(ReactiveTransactionManager transactionManager) {
        return TransactionalOperator.create(transactionManager);
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnSingleCandidate(PlatformTransactionManager.class)
    public static class TransactionTemplateConfiguration {

        @Bean
        @ConditionalOnMissingBean(TransactionOperations.class)
        public TransactionTemplate transactionTemplate(PlatformTransactionManager transactionManager) {
            return new TransactionTemplate(transactionManager);
        }

    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnBean(TransactionManager.class)
    @ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)
    public static class EnableTransactionManagementConfiguration {

        @Configuration(proxyBeanMethods = false)
        @EnableTransactionManagement(proxyTargetClass = false)
        @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false")
        public static class JdkDynamicAutoProxyConfiguration {

        }

        @Configuration(proxyBeanMethods = false)
        @EnableTransactionManagement(proxyTargetClass = true)
        @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true",
                matchIfMissing = true)
        public static class CglibAutoProxyConfiguration {

        }

    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnBean(AbstractTransactionAspect.class)
    static class AspectJTransactionManagementConfiguration {

        @Bean
        static LazyInitializationExcludeFilter eagerTransactionAspect() {
            return LazyInitializationExcludeFilter.forBeanTypes(AbstractTransactionAspect.class);
        }
    }
}
{% endhighlight %}
트랜잭션 관리에 필요한 빈들을 자동으로 설정해주는 것을 볼 수 있다.

코드를 보면 `@ConditionalOnClass(PlatformTransactionManager.class)` 에 의해 PlatformTransactionManager(트랜잭션 매니저) 가 있을 때만 설정파일이 활성화 되는 것을 알 수 있다.

`PlatformTransactionManager` 는 트랜잭션 기능을 추상화한 인터페이스이다.
{% highlight Java%}
public interface PlatformTransactionManager extends TransactionManager {

    TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
            throws TransactionException;

    void commit(TransactionStatus status) throws TransactionException;
    void rollback(TransactionStatus status) throws TransactionException;
}
{% endhighlight %}
이렇게 트랜잭션 기능을 추상화시켜 놓으면 트랜잭션을 동작시키는 기술이 바뀌었을 때 모든 코드를 변경할 필요 없이 쉽게 변경할 수 있다.

**참고로 DataSource 를 통해 커넥션을 사용하는 기능은 인터페이스가 아닌, 각 구현체에 있다.**

jdbc 나 hibernate 같은 의존성을 포함시키면 **DataSourceTransactionManager**, **HibernateTransactionManager** 같은 구현체가 `PlatformTransactionManager` 를 상속받아 구현된다.

이후 `@EnableTransactionManagement` 를 통해 선언적 트랜잭션 관리를 활성화하고 프록시를 생성하는 방식을 선택한다. (선언적 트랜잭션 관리란, @Transactional 를 통해 트랜잭션을 관리하는 것을 말한다)

**이때, 스프링은 프록시를 만들어 트랜잭션 코드를 대신 만들어준다.**

위 코드에선 @Transactional 로 한번에 처리했지만, 스프링이 만든 프록시코드에는 아마 JDBC 로 구현했던 것 처럼 rollback(), commit() 등 트랜잭션 로직에 따라 구현이 되어 있을 것이다. 그걸 그냥 프록시가 만들어주는 것이다.

결국 우리가 @Transactional 어노테이션이 붙은 메서드를 실행하면, 해당 메서드 자체가 실행되는게 아니라, 트랜잭션 처리가 된 프록시 메서드가 실행되는 셈이다.

![](https://i.imgur.com/YcriTgk.png)

코드를 보면 프록시를 생성하는 방법에는 두 가지가 있다.
- `JdkDynamicAutoProxyConfiguration`
- `CglibAutoProxyConfiguration`

하나는 JDK 방식, 나머지는 CGLIB 를 활용해 프록시를 생성하는 방법이다.

두 방식의 차이점을 간단하게 설명하자면, JDK 프록시는 타겟 클래스의 인터페이스를 기반으로 프록시를 생성한다. 만약 타겟 클래스가 인터페이스를 구현하고 있지 않다면 사용할 수 없다. 

반면, CGLIB 프록시는 타겟 클래스를 상속받아 프록시를 생성한다. JDK 프록시와 달리, 타겟 클래스가 인터페이스를 구현하고 있지 않아도 프록시 생성이 가능하며, 타겟 클래스의 서브클래스를 생성하는 방식으로 동작한다.

직접 `getClass()` 해본 결과 CGLIB 방식을 사용하고 있는 것을 확인했다. 
![](https://i.imgur.com/g3zTLoV.png)

정리하자면, @Transactional 이 동작하는 과정은 다음과 같다. 
- `@Transactional` 이 붙은 메서드가 실행됨
- CGLIB 프록시로 만든 클래스가 트랜잭션을 처리한다.
- `PlatformTransactionManager` 의 구현체가 트랜잭션 매니저로 동작한다.
	- rollback(), commit(), getTransaction() 을 구현하고 있다.
- 트랜잭션 매니저에서 DataSource 를 통해 커넥션을 연결한다.
- 사용 후 커넥션 반납

# @Transactional 의 올바른 사용
그렇다면 `@Transactional` 을 올바르게 사용하려면 어떻게 해야할까? 또 사용하면 주의해야 할 점은 무엇일까
## Service 단에서 사용
Service 단의 특성 상 여러 DAO 를 호출하여 한 번의 비즈니스 로직으로 수행하는 경우가 많다. 이러한 작업들은 일관성을 유지해야 한다(하나가 실패하면 나머지 작업들도 모두 롤백이 되어야 함).

그렇기 위해선 해당 로직들을 한꺼번에 갖고 있는 서비스 단에서 처리해야 한다. 

추가적으로 컨트롤러 단에서 사용할 경우 HTTP 요청 처리전체가 트랜잭션으로 관리되어야 하므로 비효율적이다.
## public 과 함께 선언하기
동작원리를 보면, 스프링이 @Transactional 이 붙은 메서드의 프록시를 생성하는 과정이 있다.

이때 메서드의 접근제어자가 `private` 이나 `protected` 로 선언되었다면 메서드에 접근하지 못해 프록시 생성이 불가하다.
## readOnly=true
@Transactional 의 속성 중 `readOnly` 가 있다. 데이터를 "읽기" 만 할 땐 readOnly=true 를 통해 트랜잭션을 최적화 할 수 있다. 

참고로 readOnly 속성의 default 는 false 이다.

보통 아래와 같은 방식으로 사용한다. 
{% highlight Java%}
@Service  
@Transactional(readOnly = true)  
public class transactionTest {  

    public void A() {  
        //...code
    }  

    @Transactional  
    public void B() {
        //...code  
    }  
}
{% endhighlight %}

클래스 상단에 @Transactional(readOnly=true) 을 붙이면 클래스에 포함된 메서드에 모두 적용된다. 그 중 "쓰기" 작업을 하는 메서드에만 따로 @Transactional 을 붙여주면 최적화가 가능하다.
## 체크예외 롤백 여부
트랜잭션은 언체크예외(Runtime Exception)과 Error는 롤백하지만, 체크 예외는 롤백하지 않는다(체크 예외는 컴파일 시점에 잡아내는 예외를 말함)

rollbackFor 속성을 사용하여 특정 예외 발생 시, 롤백 여부를 결정할 수 있다. 

{% highlight Java%}
@Transactional(rollbackFor = Exception.class) 
{% endhighlight %}

## 테스트 시 자동 롤백
테스트 코드에서 @Transactional 을 사용하면, 해당 테스트 메서드가 끝난 후 자동으로 롤백하는 기능이 있다. 이러한 특성 때문에 테스트 코드에서 사용 시 주의할 점이 몇 가지 있다. 

예를 들어 테스트 코드가 정상적으로 동작해도, DB 에는 데이터가 담기지 않는 것이다.

또는 일어날 수 있는 예외가 있어도, 알아서 롤백해버리기 때문에 테스트 코드에서 이를 잡아내지 못할 가능성도 있다. 

따라서 테스트코드 작성 시에는 @Transactional 을 용도에 맞게 설정하고 사용해야 한다.

### 참고
[@Transactional 바르게 알고 사용하기](https://medium.com/gdgsongdo/transactional-%EB%B0%94%EB%A5%B4%EA%B2%8C-%EC%95%8C%EA%B3%A0-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0-7b0105eb5ed6)

[Transaction Management docs](https://docs.spring.io/spring-framework/docs/4.2.x/spring-framework-reference/html/transaction.html)

[The best way to use the Spring Transactional annotation](https://yunyoung1819.tistory.com/229)



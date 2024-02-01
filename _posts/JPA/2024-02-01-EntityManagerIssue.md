---
layout: post
title: Entity Manager 의 동시성 문제 해결과정과 실험
category: JPA
excerpt: "코드에서 여러 스레드가 동시에 saveAndPrint 메소드를 호출하고 있음에도 불구하고, 각 Member는 제대로 저장되고 있다. 이는 엔티티 매니저가 각각의 트랜잭션마다 독립적으로 작동하기 때문이다. (동시성 이슈가 발생하지 않음) 또한, 결과에서 EntityManager의 클래스 이름이 jdk.proxy3.$Proxy108로 출력되는 것을 볼 수 있다. 이는 EntityManager가 실제로 프록시로 감싸져 있음을 알 수 있다.스프링컨테이너가 관리하는 엔티티 매니저 빈은 싱글톤이 맞다. 하지만 @PersistenceContext 를 통해 EntityManager 객체에 실제로 DI 되는 엔티티매니저는 프록시객체였다. 위에서 확인한 결과와"
---

엔티티 개발이 모두 끝나고 레포지토리 계층을 만들고 있었다. 어떻게 만들었는지 이전 기억을 떠올려봤다. 레포지토리는 DB와 데이터를 직접적으로 주고 받는 계층이다. 따라서 `Entity Manager` 를 사용해야 한다.(물론 `JpaRepository` 인터페이스를 상속받아 구현하는 방법도 있지만 좀 더 정밀한 제어와 JPA에 대한 이해도를 높이고 싶어 일부러 엔티티 매니저를 직접 사용했다.)

# @PersistenceContext
엔티티 매니저를 직접 사용하는 방법은 간단하다. `@PersistenceContext` 를 붙이면 된다. 
{% highlight Java%}
@Repository  
public class MemberRepository {  
	  
   @PersistenceContext  
   private EntityManager em;  
  
   //code
}
{% endhighlight %}
`@PersistenceContext` 에 대해 좀 더 자세히 설명하자면, 스프링 컨테이너로부터 `EntityManager` 를 주입받는 기능을 한다. 즉, 스프링 컨테이너가 관리하는 `EntityManager` 를 주입받게 된다. 이 방법은 `EntityManagerFactory` 를 직접 만들어 사용하는 방법과 달리, `EntityManager` 의 생명주기를 따로 관리하지 않아도 된다는 장점이 있다. 추가로 `EntityManager` 와 그에 연결된 영속성 컨텍스트가 현재 트랜잭션의 생명주기와 동일하게 관리된다.

`@Autowired` 를 통해 주입 받을 수도 있다.(생성자 주입 방식 사용하기)
{% highlight Java%}
@Repository
@RequiredArgsConstructor
public class MemberRepository {

   private final EntityManager em;
   
   //code
}
{% endhighlight %}

## EntityManager 와 EntityManagerFactory
엔티티 매니저가 DB 관련 처리를 어떻게 하는 지 구체적으로 이해하고 싶었다.

엔티티의 영속성을 보장하고 DB 와의 상호작용을 효율적으로 처리하기 위해 `영속 컨텍스트` 가 존재한다. 데이터는 DB 에 가기 전 영속 컨텍스트 내부에서 처리 된 후 DB 에 최종적으로 저장된다. 

이때 영속 컨텍스트와 DB 사이를 관리해주는 인터페이스를 `EntityManager` 라고 한다. 데이터를 담아서 옮기거나 할 땐 EntityManager 를 통해 실행해야 한다. EntityManager 는 트랜잭션마다 생성된다. 각 트랜잭션은 고유의 EntityManager를 가지며, 이러한 관계 속에서 영속성 컨텍스트와 DB 사이의 작업을 처리한다.

그리고 이러한 EntityManager 를 만드는 것이 `EntityManagerFactory` 이다. 생성비용이 매우 크기 때문에 애플리케이션 실행 시점에 딱 한번만 만들어진다. 

![](https://i.imgur.com/WK5yRwk.png)

# 엔티티 매니저의 동시성 문제
`@PersistenceContext` 를 통해 스프링 컨테이너로부터 `EntityManager`를 주입받는다. 그런데 여기서 궁금증이 생겼다. 

**"스프링 컨테이너가 관리하는 빈들은 기본적으로 "싱글톤" 이다. 엔티티 매니저가 싱글톤인 경우 동시성 문제가 발생하지 않을까?"**

그래서 직접 확인해봤다. 
{% highlight Java%}
//MemberRepository
@Repository
public class MemberRepository {

   @PersistenceContext
   private EntityManager em;

   @Transactional
   public void saveAndPrint(Member member) {
      em.persist(member);
      System.out.println("Current EntityManager: " + this.em.getClass());
      System.out.println("Thread: " + Thread.currentThread().getName() + ", Saved Member ID: " + member.getId());
   }
}
{% endhighlight %}

테스트코드에서도 5개의 쓰레드를 직접 만들었다.
{% highlight Java%}
//MemberRepositoryTest
@SpringBootTest  
class MemberRepositoryTest {  

   @Autowired  
   MemberRepository memberRepository;  

   @Test  
   void testEntityManager() throws Exception {  
      ExecutorService executorService = Executors.newFixedThreadPool(5);  
      for (int i = 0; i < 5; i++) {  
         executorService.execute(() -> {  
            Member member = new Member();  
            memberRepository.saveAndPrint(member);  
         });  
      }  
      executorService.shutdown();  
      executorService.awaitTermination(1, TimeUnit.MINUTES);  
   }
}
{% endhighlight %}
각 쓰레드에서 사용되는 엔티티매니저들은 싱글톤(=모두 같은가)인가? 만약 맞다면 동시성은 어떻게 해결하는지 궁금했다. 

실제 결과는 다음과 같다.(member ID는 @GeneratedValue 자동생성을 사용했다.)
![](https://i.imgur.com/rpwb7hp.png)
코드에서 여러 쓰레드가 동시에 `saveAndPrint` 메소드를 호출하고 있음에도 불구하고, 각 Member는 제대로 저장되고 있다. 이는 엔티티 매니저가 각각의 트랜잭션마다 독립적으로 작동하기 때문이다. (동시성 이슈가 발생하지 않음)

또한, 결과에서 `EntityManager`의 클래스 이름이 `jdk.proxy3.$Proxy108`로 출력되는 것을 볼 수 있다. 이는 `EntityManager`가 실제로 프록시로 감싸져 있음을 의미한다.

인터넷을 좀 더 찾아보고 정리한 결론은 다음과 같다.

스프링 컨테이너가 관리하는 `EntityManager` 빈은 "싱글톤"이 맞다. 하지만 `@PersistenceContext`를 통해 주입되는 `EntityManager`는 실제로는 "프록시" 객체이다. 이 프록시 객체는 모든 쓰레드에서 동일하다. 

이후, 이 프록시 객체를 통해 각 쓰레드마다 별도의 실제 `EntityManager` 인스턴스가 생성되고 사용된다. 따라서 `EntityManager`가 동시성 이슈를 일으키지 않는 이유는 프록시 객체가 각각의 쓰레드에 알맞은 실제 `EntityManager`를 할당해주기 때문이다.

# 영속 컨텍스트 유형
추가적으로, 직접 테스트를 해보면서 조금 헤맸던 부분이다. 엔티티매니저 프록시가 엔티티 매니저들을 각각의 트랜잭션에 맞게 알맞은 영속컨텍스트로 할당시켜주는 것은 이해했다. 문제는 영속 컨텍스트가 언제 어떻게 생성되는가 였다.

JPA에서 제공하는 영속 컨텍스트는 두 가지가 있다. 
-  `Transaction-scoped persistence context`
-  `Extended-scoped persistence context` 

간단하게 설명하자면, `Transaction-scoped persistence context` 은 트랜잭션 하나를 기준으로 생성되는 영속 컨텍스트이고, `Extended-scoped persistence context` 여러 트랜잭션에 걸쳐 생성되는 영속 컨텍스트이다. 

보통 Transaction-scoped persistence context 사용을 권장한다. 

`@Transactional` 어노테이션을 사용하면, 해당 메서드의 트랜잭션 범위 내에서 EntityManager가 동작하게 된다. 
이 EntityManager는 `Transaction-scoped persistence context`를 사용하므로, 트랜잭션의 시작과 종료에 맞춰 영속컨텍스트가 생성, 종료된다. 테스트 시에는 `@Transactional` 어노테이션을 붙여 사용하자.

{% highlight Java%}
@SpringBootTest  
class MemberRepositoryTest {  

   @Autowired  
   MemberRepository memberRepository;  

   @Test  
   @Transactional  
   void memberSave() throws Exception {  
      //test code
   }
}
{% endhighlight %}

### 참고
[인프런 질문](https://www.inflearn.com/questions/158967/%EC%95%88%EB%85%95%ED%95%98%EC%84%B8%EC%9A%94-entitymanager%EC%97%90-%EB%8C%80%ED%95%B4-%EA%B6%81%EA%B8%88%ED%95%9C-%EC%A0%90%EC%9D%B4-%EC%9E%88%EC%96%B4-%EC%A7%88%EB%AC%B8-%EB%82%A8%EA%B9%81%EB%8B%88%EB%8B%A4)

[JPA/Hibernate Persistence Context](https://www.baeldung.com/jpa-hibernate-persistence-context)

[EntityManagerFactory, EntityManager 그리고 PersistenceContext](https://iyoungman.github.io/jpa/EntityManagerFactory-EntityManager-PersistenceContext/)

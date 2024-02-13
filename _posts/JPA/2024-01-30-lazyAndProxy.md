---
layout: post
title: 지연로딩을 통한 성능최적화 + 프록시에 대한 구체적 이해
category: JPA
excerpt: "테이블 상으론 `Diet` 가 `Member` 의 FK를 갖고 있지만, 객체 관점에선 엔티티 자체를 참조하고 있다. 이때 `Diet` 를 통해 `Member` 의 필드를 갖고 오려면 `Member` 의 정보를 이전에 미리 모두 참조 해야 하는 것 아닌가? 실제론 미리 등록하지 않고 필요할 때마다 갖고 오는 지연로딩은 구체적으로 어떻게 구현되는 건지에 대한 의문이 생겼다.프록시가 지연로딩을 구현하는 방법은 다음과 같다."
---

 `Member` 와 `Diet`. 각각 "회원"과 "식단"을 가리키는 두 엔티티가 있다. 두 엔티티는 일대다, 다대일로 **연관관계 매핑**이 되어 있다. 
테스트 코드를 작성하던 중 연관관계에 있는 두 엔티티가 호출될 때 문제를 발견했다.

{% highlight Java%}
@SpringBootTest  
public class MemberDietTest {  
  
  @Autowired  
  EntityManager em;  
	  
  @Test  
  @Transactional  
  void memberDietTest() throws Exception {  
	
	  Member member = new Member("MemberA");  
	  Diet diet = new Diet("dietA");  
	  member.addDiet(diet);  
		
	  em.persist(member);  
	  em.persist(diet);
		
	  em.flush();  
	  em.clear();  
		  
	  System.out.println("==== diet ====");  
	  Diet findDiet = em.find(Diet.class, diet.getId());  
	  System.out.println("==== member ====");  
	  String memberName = findDiet.getMember().getUsername();  
	  System.out.println("memberId = " + memberName);
	}  
}
{% endhighlight %}
문제를 좀 더 구체적으로 설명하자면, 
Diet를 조회하는 시점에 Diet 와 연관관계에 있는 Member 까지 모두 가져왔다.
{% highlight Java%}
Diet findDiet = em.find(Diet.class, diet.getId());  
{% endhighlight %}

그리고 Member의 이름을 조회하는 시점에 Member를 한번 더 SELECT 했다. 
{% highlight Java%}
String memberName = findDiet.getMember().getUsername();
{% endhighlight %}

![](https://i.imgur.com/7bL8IlX.png)
"\=\=\=\= member \=\=\=\=" 이후에 SELECT 쿼리가 나가지 않고, **Diet가 조회될 때 Member가 함께 SELECT** 되는 것을 볼 수 있다. 

이렇게 되면 Diet 엔티티가 연관관계를 많이 가질 경우(Member 도 마찬가지) 당장 사용하지 않는 엔티티들까지 모두 SELECT 하면서 성능이 저하될 우려가 있다. 연관관계에 있는 해당 엔티티를 실제 사용할 때 SELECT 하는 것이 좋을 것이다.

# 지연로딩
"지연로딩"을 이용하면 해결할 수 있다. 코드부터 보면, @ManyToOne, @OneToMany 같은 연관관계를 맺은 어노테이션에 `fetch = FetchType.LAZY` 으로 설정해주면 된다.

{% highlight Java%}
@Entity  
public class Diet {  
  
	@Id  
	@GeneratedValue  
	@Column(name = "diet_id")  
	private Long id;  
	  
	private String dietName;  
	  
	@ManyToOne(fetch = FetchType.LAZY) //LAZY 설정 
	@JoinColumn(name = "member_id")  
	private Member member;
}
{% endhighlight %}

{% highlight Java%}
@Entity  
public class Member {  
  
	@Id  
	@GeneratedValue  
	@Column(name = "member_id")  
	private Long id;  
	  
	private String username;  
	  
	@OneToMany(mappedBy = "member", fetch = FetchType.LAZY) //LAZY 설정
	private List<Diet> diets = new ArrayList<>();
}
{% endhighlight %}

지연로딩으로 바꾼 후 다시 실행해보면 `Diet`가 SELECT 될 때, `Member` 는 SELECT 되지 않는다. 그리고 "\=\=\=\= member \=\=\=\=" 이후 실제 `Member` 객체가 사용될 때 SELECT 되는 것을 확인할 수 있다.
![](https://i.imgur.com/XMJgYQU.png)
## 어떻게 가능한거지?
이렇게 지연로딩으로 간단히 설정을 바꿈으로써 성능최적화를 해보았다. 이때 궁금한 점이 생겼다. 

테이블 상으론 `Diet` 가 `Member` 의 FK를 갖고 있지만, 객체 관점에선 엔티티 자체를 참조하고 있다. 이때 `Diet` 를 통해 `Member` 의 필드를 갖고 오려면 `Member` 의 정보를 이전에 미리 모두 참조 해야 하는 것 아닌가? 

실제론 미리 등록하지 않고 필요할 때마다 갖고 오는 지연로딩은 구체적으로 어떻게 구현되는 건지에 대한 의문이 생겼다.
# 프록시
JPA Hibernate(하이버네이트)는 **프록시**를 통해 지연로딩을 구현한다. 프록시의 의미 자체는 "대리", "대행자" 라는 의미이다. 구현 방법도 비슷한 느낌이다.

프록시가 지연로딩을 구현하는 방법은 다음과 같다. 
`Member` 엔티티의 '가짜'를 만든다(이게 프록시 객체). 

이 프록시는 실제 `Member` 엔티티와 동일한 인터페이스를 가지지만, 내부적으로 실제 `Member` 엔티티의 데이터는 포함하지 않는다. Diet 엔티티는 **Member 프록시를 참조**한다.

`Diet` 엔티티가 처음으로 `Member` 엔티티의 데이터에 접근하려고 할 때, 프록시는 그제서야 실제 `Member` 엔티티를 데이터베이스에서 로드한다(프록시 초기화). 

**프록시가 실제 객체처럼 동작할 수 있는 이유는 실제 객체를 상속하고 있기 때문이다.** 또한 실제 객체를 가리키는 속성이 있다. 프록시는 실제 객체의 참조를 보관하고 있다가 호출 시, 실제 객체를 반환한다.

주의할 점)
- 프록시 객체는 처음 사용할 때 한 번만 초기화
- 프록시 객체를 초기화 할 때, 프록시 객체가 실제 엔티티로 바뀌는 것은 아님. 초기화되면 프록시 객체를 통해서 실제 엔티티에 접근 가능
- 단지, 프록시는 실제 객체를 반환해주는 것 뿐임

이렇게 실제로 필요한 경우에만 `Member` 엔티티를 로드하게 되어, 불필요한 데이터베이스 쿼리를 줄이는 방식이다.

{% highlight Java%}
System.out.println("==== diet ====");  
Diet findDiet = em.find(Diet.class, diet.getId());  
Member findMember = em.getReference(Member.class, member.getId());  
System.out.println("findMember = " + findMember.getClass());
{% endhighlight %}
`getRegerence()` 는 실제 객체의 프록시를 반환하는 메서드다.
위의 예제에서 `Diet` 엔티티를 find 한 이후, `Member` 엔티티가 프록시 객체로 등록되어 있는 것을 확인할 수 있다. 

![](https://i.imgur.com/2zHL3md.png)

## 두 엔티티 간의 타입 비교는 instanceof 
두 객체의 타입이 같은 지 다른 지 비교할 땐 "\=\=" 을 사용할 수 있다.
{% highlight Java%}
Member member1 = new Member();  
Member member2 = new Member();  

System.out.println("member1 == member2: " + (member1.getClass() == member2.getClass())); //true
{% endhighlight %}

하지만 지연로딩을 사용한 경우, 비교대상의 객체가 프록시 일 수 있다. 따라서 "\=\=" 을 그대로 사용하는 것은 위험하다. 

왜냐하면 프록시는 실제 객체를 상속받은 것이지, 엄연히 다른 타입이기 때문이다.
{% highlight Java%}
Member findMember1 = em.getReference(Member.class, member1.getId());  
Member findMember2 = em.getReference(Member.class, member2.getId());  

System.out.println("findMember1 = findMember2: "+(findMember1 == findMember2)); //false
{% endhighlight %}

따라서 instanceof 를 사용해야 정확한 타입 비교가 가능하다. 
{% highlight Java%}
System.out.println("findMember1: "+(findMember1 instanceof Member)); //true
{% endhighlight %}

지금처럼 하나의 예제에서 보면 되게 쉬워보인다. 근데 메서드 단위로 분리되면 정말 헷갈릴 수 있다고 느꼈다. 

`equals()` 메서드를 오버라이드 한다고 해보자. 파라미터로 들어오는 객체가 실제 객체를 상속받은 프록시 객체라면 `equals()` 내부에서 `instanceof` 를 통해 구현해야 한다. 이러한 프록시 객체는 잘 보이지도 않아서 충분히 실수할 여지가 있다.
{% highlight Java%}
Member member = new Member();
Member findMember2 = em.getReference(Member.class, member.getId());  
{% endhighlight %}

# 영속 컨텍스트에 이미 실제 엔티티가 있다면?
지금까지의 예제는 `Diet` 엔티티의 연관관계에 있는 `Member` 엔티티가 프록시 객체로써 어떻게 동작하는 지 살펴봤다. 

그렇다면 `Diet` 도 마찬가지로 프록시가 존재할까? 이전의 예제를 다시 가지고 왔다.
{% highlight Java%}
em.find(Diet.class, diet.getId(); //실제 엔티티 조회 
Diet reference = em.getReference(Diet.class, diet.getId()); //프록시
System.out.println(reference.getClass()); //실제 객체 반환
{% endhighlight %}

놀랍게도 프록시가 존재하지 않는다. 심지어 `getReference()` 메서드를 사용했음에도 실제 객체가 반환되는 것을 확인했다. 
![](https://i.imgur.com/ZxIQir0.png)

여기서 더 재미있는 것은 만약 프록시가 먼저 초기화된 이후, 실제 객체를 조회하면, 해당 객체의 타입은 `Proxy` 로 반환 된다.
{% highlight Java%}
em.getReference(Diet.class, diet.getId()); //프록시 초기화
Diet findDiet = em.find(Diet.class, diet.getId()); //실제 객체 조회
System.out.println(findDiet.getClass()); //프록시 반환
{% endhighlight %}

![](https://i.imgur.com/8fGBSkc.png)

이는 영속컨텍스트의 `동일성 보장`  이라는 특성 때문이다. 자바 컬렉션에서 값을 가져오면 같은 주소의 값을 가져오듯이, 영속컨텍스트 내에서 같은 PK 값을 가진 객체들을 가져올 시 `동일성` 을 보장해준다.

### 참고
자바 ORM 표준 JPA 프로그래밍 (김영한 저)

[JPA Hibernate 프록시 제대로 알고 쓰기](https://tecoble.techcourse.co.kr/post/2022-10-17-jpa-hibernate-proxy/)

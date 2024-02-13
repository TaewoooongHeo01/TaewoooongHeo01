---
layout: post
title: 값 타입 컬렉션. 왜, 언제, 어떻게 사용하는가?
category: JPA
excerpt: "이런 현상이 발생하는 이유는 값 타입 컬렉션의 특성 때문이다. 값 타입 컬렉션은 엔티티와 달리 식별자(PK)가 따로 존재하지 않는다. 따라서 JPA는 값 타입 컬렉션을 추척할 수 없다. 그래서 모두 DELETE 하고 다시 INSERT 하는 것이다. 그렇다면 이걸 어떻게 극복할까? 엔티티로 만들어 사용하면 된다. 물론 무조건 엔티티로 만들어 사용하라는 것은 아니다. 엔티티로 만들어 사용하는 것을 고려해야 한다는 것이다. Food 에 Nutri 가 값 타입 컬렉션으로 있던 기존의 예제에서 NutriEntity 를 추가했다."
---

프로젝트를 진행하면서 값 타입을 생각보다 많이 사용했다. 그런데 중간중간 헷갈리는 부분들이 있었다. 잘못 사용하지 않도록 확실하게 알아야겠다고 느껴 차근차근 정리해봤다.
# 값 타입
데이터 타입은 크게 **값 타입**과 **엔티티 타입**으로 분류할 수 있다. 

엔티티타입은 `@Entity` 로 정의한다. 엔티티는 DB의 테이블과 직접 매핑된다. 또한 데이터가 변경되도 추적할 수 있다. 예를 들어 회원 엔티티에서 나이가 변해도 여전히 같은 회원 엔티티이다.

값 타입은 int, Integer 같은 단순히 값을 저장하는 타입을 말한다. 데이터 변경 시 추척할 수 없다. 
{% highlight Java%}
int a = 10;
int b = a;

a = 5;
System.out.println(b); //10
{% endhighlight %}
b에 a를 복사했다. a의 데이터가 변경되어도 b는 변경되지 않는다. 

여기서 값 타입이 `불변성(Immutability)`을 가진다는 것을 알 수 있다. 즉, 값 타입의 인스턴스는 한 번 생성되면 그 상태를 변경할 수 없다. 이는 값 타입이 메모리가 아닌 값 자체만 복사되므로, 원본 데이터가 변경되어도 복사된 데이터는 변경되지 않는다는 특성 때문이다.

값 타입에는 primitive 타입(int, double 등), Wrapper 클래스(Integer, Long 등), 임베디드 타입, 컬렉션 값 타입 등이 포함된다. 또한 **식별자를 갖지 않는다.** 

참고로 주의할 점은 값 타입이 클래스인 경우(임베디드 타입) 참조에 의한 복사가 되므로 불변성을 가지지 못할 수 있다. 이를 주의해야 한다. 
# 값 타입 컬렉션이란?
**값 타입 컬렉션이란 값 타입을 자바 컬렉션에 담아 사용하는 것이다.** 엔티티들끼리 일대다 매핑 후, 컬렉션에 담아 사용하는 것 비슷한 맥락이다.

값 타입 컬렉션은 엔티티의 일부로서 엔티티의 생명주기를 따르며, 엔티티 없이는 존재할 수 없다.

음식 엔티티에 "영양성분"과 "알러지 정보"가 있다고 하자. 영양성분과 알러지정보 모두 값 타입으로 만들었으며, 음식엔티티에 여러개가 존재할 수 있다고 가정한다.
![](https://i.imgur.com/oVmXTXJ.png)

문제는 자바 클래스를 테이블에 매핑할 때 발생한다. 기본적으로 DB 테이블에는 컬렉션을 담을 수 없다. 

음식의 입장에선 어쨌든 알러지정보와 영양정보가 여러 개 갖고 있는 건 맞다. 따라서 별도의 테이블을 만들어 `일대다` 로 풀어내야 한다.
![](https://i.imgur.com/GnJUAdQ.png)
여기서 Allergy와 Nutri클래스의 필드 값들이 모두 PK인 이유는 이 클래스들이 모두 `값 타입` 이기 때문이다. 값 타입이므로 자체적으로 식별자를 가지지 않으며, 따라서 엔티티의 일부로서 엔티티의 식별자를 사용한다.

{% highlight Java%}
@Entity  
@Getter  
@Setter  
public class Food {  
	  
	@Id  
	@GeneratedValue  
	@Column(name = "food_id")  
	private Long id;  
	  
	private String foodName;  
	  
	@ElementCollection  
	@CollectionTable(  
            name = "nutri",  
            joinColumns = @JoinColumn(name = "food_id"))  
	private List<Nutri> nutriList = new ArrayList<>();  
	  
	@ElementCollection  
	@CollectionTable(  
            name = "allergy",  
            joinColumns = @JoinColumn(name = "food_id"))  
	@Column(name = "allergy")  
	private List<String> allergyList = new ArrayList<>();
}
{% endhighlight %}
- `@ElementCollection`:값 타입 컬렉션과 매핑할 때 사용한다. JPA가 해당 필드를 값 타입으로 인식할 수 있다. 
- `@CollectionTable`: 값 타입 컬렉션을 매핑한 테이블의 정보를 설정한다.

이후 다음과 같이 데이터를 직접 넣고 실행해봤다. 
{% highlight Java%}
Food food = new Food();  
food.getAllergyList().add("땅콩");  
food.getAllergyList().add("우유");  
  
food.getNutriList().add(new Nutri("1회 제공량: 150g", "136kcal", "100g", "20g", "30g"));  
food.getNutriList().add(new Nutri("1회 제공량: 100g", "100kcal", "80g", "15g", "23g"));  

em.persist(food);
{% endhighlight %}

DB 에 성공적으로 담긴다. `FOOD_ID` 도 올바르게 매핑되었다.
![](https://i.imgur.com/2x6d05L.png)

쿼리도 잘 INSERT 된다.
![](https://i.imgur.com/UyxC2Wp.png)

![](https://i.imgur.com/ZZ8HaC4.png)

![](https://i.imgur.com/OqwrdAx.png)

# 값 타입 컬렉션의 특징
위 결과로부터 우리는 값 타입 컬렉션의 특징들을 유추할 수 있다. 
## 영속성 전이
실행한 코드를 다시 보면 food 만 `persist()` 했다. 값 타입인 Allergy와 Nutri는 따로 persist 하지 않은 것을 확인할 수 있다.

하지만 두 값 타입 모두 제대로 저장되었다.

{% highlight Java%}
Food food = new Food();  
food.getAllergyList().add("땅콩");  
food.getAllergyList().add("우유");  
  
food.getNutriList().add(new Nutri("1회 제공량: 150g", "136kcal", "100g", "20g", "30g"));  
food.getNutriList().add(new Nutri("1회 제공량: 100g", "100kcal", "80g", "15g", "23g"));  

em.persist(food);
{% endhighlight %}
이것이 의미하는 바는 값 타입 컬렉션은 영속성 전이가 된다는 것이다. 즉, 엔티티가 persist 되면, 해당 엔티티에 포함된 값 타입 컬렉션들도 자동으로 persist 가 된다. 

이것은 **값 타입이 엔티티의 생명주기에 의존하기 때문**이다. 값 타입은 엔티티 없이는 존재할 수 없으며, 엔티티의 상태 변화에 따라 값 타입도 함께 변화한다.

## 지연로딩
예제코드를 다음과 같이 살짝 바꿔보았다.
{% highlight Java%}
Food food = new Food();  
food.getAllergyList().add("땅콩");  
food.getAllergyList().add("우유");  
  
food.getNutriList().add(new Nutri("1회 제공량: 150g", "136kcal", "100g", "20g", "30g"));  
food.getNutriList().add(new Nutri("1회 제공량: 100g", "100kcal", "80g", "15g", "23g"));  
  
em.persist(food);  
  
em.flush();  
em.clear();  
  
Food findFood = em.find(Food.class, food.getId());  
System.out.println("allergy: "+findFood.getAllergyList().get(0));  
System.out.println("nutri: "+findFood.getNutriList().get(0));  
{% endhighlight %}
persist 이후, 영속컨텍스트를 flush 하고, 1차 캐시를 clear 했다. 이후 DB 에서 `food` 를 찾았다. 

결과는 아래와 같다. 
![](https://i.imgur.com/NWeFTAm.png)
-> Food findFood = em.find(Food.class, food.getId());  <br>
food 를 가져올 때는 Allery와 Nutri를 가져오지 않는다.

![](https://i.imgur.com/ApsFl8A.png)
-> System.out.println("allergy: "+findFood.getAllergyList().get(0)); <br>
allery 를 필요로 할 때 그제서야 `SELECT` 쿼리가 나가는 것을 볼 수 있다. 

![](https://i.imgur.com/8VwSMDD.png)
-> System.out.println("nutri: "+findFood.getNutriList().get(0)); <br>
Nutri도 마찬가지다.

이 결과를 통해 값 타입 컬렉션은 `지연로딩` 인 것을 알 수 있다. 

실제로 `@ElementCollection` 의 fetch 속성 default 는 `LAZY` 이다.
![](https://i.imgur.com/6xfkCiV.png)

지연로딩이란 불필요한 쿼리를 막기 위해 해당 데이터를 실제 사용할 때 불러오는 방식을 말한다. 이전에 자세하게 정리한 내용이 있다. <br>
-> [지연로딩을 통한 성능최적화 + 프록시에 대한 구체적 이해](https://taewoooongheo01.github.io/TaewoooongHeo01/jpa/2024/01/30/lazyAndProxy/)

# 값 타입 컬렉션 수정
지금까지의 예제는 "조회"만 했다. 이번엔 "수정"을 해볼것이다. **중요한 점은 값 타입은 불변해야 한다는 것**이다. 따라서 setter 만을 이용해서 값을 수정하는 건 안된다. setter 로 수정하는 경우 `side effect` 가 발생할 수 있다. <br>
-> 예를 들어 여러 개의 객체가 하나의 값 타입을 참조하고 있는 경우 모든 엔티티의 값 타입이 변경되는 일이 일어난다.

값 타입을 수정할 땐 아예 삭제하고 하나를 다시 만들자. <br>
아래 예제에선 알러지 목록에 "땅콩"을 삭제하고 "달걀"을 추가했다. 
{% highlight Java%}
Food findFood = em.find(Food.class, food.getId());  
findFood.getAllergyList().remove("땅콩");  
findFood.getAllergyList().add("달걀");
{% endhighlight %}
단순 String인 Allergy의 경우 간단하게 `remove()` 하고 `add()` 했다.

값 타입이 임베디드 타입인 경우도 똑같다. 해당 클래스의 참조를 직접 찾아서 없애고 수정된 사항을 `new` 로 만들어 add 한다. 아래는 "1회 제공량" 부분만 수정한 예제다.
{% highlight Java%}
//일치하는 Nutri 클래스 찾아 삭제
Nutri targetNutri = null;  
for (Nutri nutri : findFood.getNutriList()) {  
	if (nutri.getServingSize().equals("1회 제공량: 150g") &&  
            nutri.getTotalCalorie().equals("136kcal") && 
	    nutri.getCarbohydrate().equals("100g") && 
	    nutri.getProtein().equals("20g") && 
	    nutri.getFat().equals("30g")) {  
                targetNutri = nutri;  
                break;  
	}  
}  

if (targetNutri != null) {  
	findFood.getNutriList().remove(targetNutri);  
}  

//수정된 클래스 추가
findFood.getNutriList().add(new Nutri("1회 제공량: 200g", "136kcal", "100g", "20g", "30g"));
{% endhighlight %}
결국엔 삭제하고 아예 새로 만들어야 한다. 하지만 우리가 주목해야 할 건 쿼리다.

![](https://i.imgur.com/2b0iBcf.png)
쿼리를 잘 보면 `DELETE` 이후 `INSERT` 가 두 번 나가는 것을 확인했다. 나는 분명 하나만 삭제하고 하나만 추가했는데 이게 어떻게 된 걸까?

이것이 바로 값 타입 컬렉션의 한계다. 값 타입 컬렉션이 수정될 시, 주인 엔티티와 연관된 다른 값 타입들을 모두 DELETE 하고 다시 INSERT 한다. 수정한 값 타입만 DELETE 했다가 INSERT 할 줄 알았는데 아니었다. 

그리고 이것은 다음과 같은 문제들을 일으킬 수 있다.
- 불필요한 쿼리로 인한 성능 문제
- 동시에 여러 트랜잭션이 같은 컬렉션을 수정할 시 컬렉션의 일관성이 깨질 수 있는 문제

# 한계 극복
이런 현상이 발생하는 이유는 값 타입 컬렉션의 특성 때문이다. 

값 타입 컬렉션은 엔티티와 달리 `식별자(PK)`가 따로 존재하지 않는다. 따라서 JPA는 값 타입 컬렉션을 추척할 수 없다. 그래서 모두 DELETE 하고 다시 INSERT 하는 것이다. 

그렇다면 이걸 어떻게 극복할까? `엔티티` 로 만들어 사용하면 된다. 물론 무조건 엔티티로 만들어 사용하라는 것은 아니다. 엔티티로 만들어 사용하는 것을 고려해야 한다는 것이다. 

Food 에 Nutri 가 값 타입 컬렉션으로 있던 기존의 예제에서 `NutriEntity` 를 추가했다. 
{% highlight Java%}
@Entity  
@Table(name = "nutri")  
public class NutriEntity {  
  
	@Id  
	@GeneratedValue  
	private Long id;  

	@Embedded
	private Nutri nutri;  
	  
	public NutriEntity() {};  
	  
	public NutriEntity(String s, String s1, String s2, String s3, String s4) {  
		this.nutri = new Nutri(s, s1, s2, s3, s4);  
	};  
}
{% endhighlight %}
`Food` 엔티티와 `Nutri` 값 타입 중간에 `NutriEntity` 를 만들어 매핑한다. 

{% highlight Java%}
//Food

// @ElementCollection  
// @CollectionTable(  
// name = "nutri",  
// joinColumns = @JoinColumn(name = "food_id"))  
// private List<Nutri> nutriList = new ArrayList<>();  
  
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)  
@JoinColumn(name = "food_id")  
private List<NutriEntity> nutriList = new ArrayList<>();
{% endhighlight %}
Food 엔티티는 NutriEntity 와 매핑했기 때문에 `@OneToMany` 일대다로 매핑한다. 

![](https://i.imgur.com/9Kzo47O.png)
이후 결과를 보면 이전과 달리 `Nutri` 객체에 ID 식별자가 생긴 것을 확인할 수 있다. 이러면 따로 수정이 가능해진다(추적이 가능하기 때문).

# 결론
값 타입 컬렉션은 수정 과정에서 발생하는 성능 저하와 데이터 일관성 문제가 있다.

따라서, 가능하다면 엔티티를 사용하는 것을 고려하자. 특히, 해당 도메인을 따로 조회하거나 업데이트 해야하는 경우, 그것은 값 타입이 아닌 엔티티로 관리하는 것이 더 적합하다.

엔티티의 생명주기와 동일하게 따라가는, 정말 간단한 값들만 값 타입으로 만들자. 

값 타입 컬렉션을 다시 공부하면서 값 타입의 정확한 용도와 한계를 알았다. 프로젝트에 있는 값 타입들 몇 개는 엔티티로 수정해야겠다. 

### 참고
자바 ORM 표준 JPA 프로그래밍 - 김영한

[JPA: When to choose Multivalued Association vs. Element Collection Mapping](https://stackoverflow.com/questions/3418683/jpa-when-to-choose-multivalued-association-vs-element-collection-mapping)


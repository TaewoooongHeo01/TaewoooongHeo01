---
layout: post
title: 다대다[M:N] 극복하기
category: JPA
excerpt: "`@ManyToMany` 는 함정이 있다. 바로 필드가 추가되어야 하는 경우에 필드를 추가할 수 없다는 것이다. 따라서 `@ManyToMany` 는 쓰지 않는게 좋다. 그렇다면 @ManyToMany는 아예 쓰면 안되는걸까? 처음엔 명백히 필드가 추가될 가능성이 없다면 그냥 써도 되겠다고 생각했다. 하지만 `@ManyToMany` 는 추가적인 문제가 여럿 있었다.대표적으론 성능과 관련한 문제가 있다. 
`@ManyToMany` 를 통해 중간테이블을 자동생성한 경우, 중간테이블을 JPA가 관리하게 된다."
---

이번에 엔티티 설계를 처음으로 직접 해봤다. 직접 해보려니 헷갈리는 것들이 꽤나 있었다. 

그 중에서도 다대다 관계가 특히 헷갈렸다. 물론 처음 만들어본 거라 헷갈리는게 당연하지만 앞으로 계속해서 만들 엔티티연관관계이고, 꽤 중요하다고 생각해 정리를 해봤다. 

# 다대다 [M:N]
나는 현재 식단을 만들어주는 서비스를 개발하고 있다. 이 서비스에는 두 엔티티가 등장한다. 

- 식단
- 음식

이 두 엔티티는 다음 조건을 만족한다. 
- "식단에는 여러 음식이 들어갈 수 있다" 
- "하나의 음식은 여러 식단에 속할 수 있다"

위 두 조건을 엔티티간의 연관관계로 해석하면 **다대다\[M:N]** 관계가 된다.

이러한 관계는 객체 입장에서 쉽게 구현가능하다. 그냥 서로의 리스트를 가지면 된다.
{% highlight java%}
public class Diet {  
  
	@Id  
	@GeneratedValue  
	@Column(name = "DIET_ID")  
	private Long id;  
	  
	private List<Food> dietfoods = new ArrayList<>();
}
{% endhighlight %}

{% highlight java%}
public class Food {  
	  
	@Id  
	@GeneratedValue  
	@Column(name = "FOOD_ID")
	private Long id;  
	  
	private List<Diet> foodDiets = new ArrayList<>();
}
{% endhighlight %}

그런데 관계형 DB 입장에서 보면? 문제가 생겼다. 관계형 DB는 다대다를 바로 나타낼 수 없다. 왜냐하면 다대다 관계를 직접 표현하려면 한 행이 여러 행을 참조하고, 동시에 여러 행이 해당 행을 참조해야 하기 때문이다. 이는 2차원 테이블을 넘어서는 형태다.

그렇다면 어떻게 해야할까? **다대다 = 일대다 + 다대일** 로 풀어내면 된다. 즉, 두 엔티티의 FK를 모두 갖는 중간테이블을 만들면 된다. 
![](https://i.imgur.com/z5V0AzZ.png)
관계형 DB는 이렇게 중간테이블을 추가하는 방법으로 다대다관계를 풀어낸다.
## @ManyToMany
다대다 관계는 `@ManyToMany` 어노테이션을 사용할 수 있다. 
{% highlight java%}
public class Diet {  
  
	@Id  
	@GeneratedValue  
	@Column(name = "diet_id")  
	private Long id;  
	  
	@ManyToMany  
	@JoinTable(name = "dietFood",  
	  joinColumns = @JoinColumn(name = "diet_id"),  
	  inverseJoinColumns = @JoinColumn(name = "food_id")  
	)  
	private List<Food> foods = new ArrayList<>();
}
{% endhighlight %}
`@JoinTable`이 바로 중간테이블을 만들어주는 어노테이션이다. 이후 중간테이블의 컬럼에 두 엔티티의 FK 를 추가하는 것을 확인할 수 있다.

{% highlight java%}
public class Food {  
  
	@Id  
	@GeneratedValue  
	private Long id;  
	  
	@ManyToMany(mappedBy = "foods")  
	private List<Diet> diets = new ArrayList<>();  
}
{% endhighlight %}
Food 엔티티에서는 mappedBy를 통해 연관관계의 주인이 Diet임을 명시했다. 

결과를 보면 중간 테이블이 생성된 것이 보인다
![](https://i.imgur.com/vAKA0vT.png)

DB에서 직접 확인해도 잘 된다.
![](https://i.imgur.com/4KWhTE3.png)

### 그치만 쓰지 마라 -> @ManyToMany
`@ManyToMany` 는 함정이 있다. 바로 필드가 추가되어야 하는 경우에 필드를 추가할 수 없다는 것이다. 예를 들어 DietFood 엔티티에 **식단에 들어갈 Food의 수량** 컬럼이 추가된다고 해보자. 이때 `@ManyToMany` 는 필드 추가가 불가하다.

실무에서 다대다는 두 엔티티간의 단순 연결로 끝나지 않는다. 생성시간 등등 추가되어야 하는 정보들이 많다. 따라서 `@ManyToMany` 는 쓰지 않는게 좋다. 

## @ManyToMany = @OneToMany + @ManyToOne
`@ManyToMany`를 쓰지 못한다면? 그냥 중간테이블을 직접 만들고 나머지 두 엔티티의 연관관계를 각각 `@OneToMany`, `@ManyToOne`으로 연결해주면 된다. 이렇게 중간테이블을 직접 만드는 경우 원하는 필드를 직접 추가할 수도 있고 확장에 유리해진다.
{% highlight java%}
public class Diet {  
	  
	@Id  
	@GeneratedValue  
	@Column(name = "diet_id")  
	private Long id;  
	
	@OneToMany(mappedBy = "diet")  
	private List<DietFood> dietfoods = new ArrayList<>();
}
{% endhighlight %}

{% highlight java%}
public class DietFood {  
  
	@Id  
	@GeneratedValue  
	private Long id;  
	  
	@ManyToOne  
	@JoinColumn(name = "diet_id")  
	private Diet diet;  
	  
	@ManyToOne  
	@JoinColumn(name = "food_id")  
	private Food food;  
}
{% endhighlight %}

{% highlight java%}
public class Food {  
  
	@Id  
	@GeneratedValue  
	@Column(name = "food_id")
	private Long id;  
	  
	@OneToMany(mappedBy = "food")  
	private List<DietFood> foodDiets = new ArrayList<>();
}
{% endhighlight %}

마찬가지로 잘 된다.
![](https://i.imgur.com/vCmQ1va.png)

### 그렇다면 @ManyToMany는 아예 쓰면 안되는걸까?
**처음엔 명백히 필드가 추가될 가능성이 없다면 그냥 써도 되겠다고 생각**했다. 하지만 `@ManyToMany` 는 추가적인 문제가 여럿 있었다.

대표적으론 성능과 관련한 문제가 있다. 
`@ManyToMany` 를 통해 중간테이블을 자동생성한 경우, 중간테이블을 JPA가 관리하게 된다. 이 경우, 개발자는 이 테이블에 대한 제어가 제한적이다. 즉, JPA가 제공하는 기능과 메서드를 사용하여 데이터를 조회하거나 조작해야 한다. 또한 조인이 자동으로 연속 실행되어, 데이터베이스 성능저하 이슈가 발생할 수 있다.

반면에, 중간 테이블을 직접 만들면 개발자가 이 테이블에 대한 완전한 제어권을 가진다. 이 경우, 특정 음식이 포함된 식단을 찾을 때, 해당 음식에 해당하는 행만 조회하면 되므로, 전체 테이블을 스캔하지 않고도 원하는 정보를 빠르게 가져올 수 있다. 
{% highlight SQL%}
SELECT d.*
FROM Diet d
INNER JOIN DietFood df ON d.id = df.diet_id 
WHERE df.food_id = :food_id
{% endhighlight %}

### 참고
[다대다 연관관계](https://seriouskang.tistory.com/7)

[만렙 개발자 키우기 - (4)다대다](https://www.nowwatersblog.com/jpa/ch6/6-4)

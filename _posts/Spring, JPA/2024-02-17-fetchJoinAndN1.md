---
layout: post
title: 페치 조인(fetch join)의 필요성 - EAGER, LAZY, JOIN 그리고 N+1 문제
category: Spring
excerpt: "N+1 문제를 통해 페치 조인(fetch join)이 왜 필요한지 이해하고, 개인적으로 헷갈렸던 EAGER, LAZY, JOIN, JOIN FETCH 의 정확한 차이점을 정리하고자 쓴 글이다. JPQL은 즉시로딩 그런거 모른다. 만약 쿼리를 'select m from Member m' 라고 짰다면, DB 에서 Member 만을 SELECT 한다. 하지만, 이때 즉시로딩으로 설정되어 있으므로 JPA에서 추가적인 SELECT 쿼리가 'select m from Member m' 시점에 같이 나간다. 즉, JOIN을 하지 않고 같은 시점에 한번 더 SELECT를 하기 때문에 N+1 문제가 해결되지 않는게 맞다. 로딩전략을 바꾸는 것 만으로는 N+1 문제를 해결할 수 없음을 알았다."
---

이 글은 N+1 문제를 해결하기 위해 시도했던 EAGER, LAZY, JOIN, JOIN FETCH 의 정확한 차이점을 정리하고자 쓴 글이다. 이를 통해 페치 조인(fetch join)이 왜 필요한 지 이해하는 것이 목적이다.

구체적으론 왜 EAGER, LAZY, JOIN 은 N+1 문제를 해결할 수 없는지, JOIN FETCH 는 왜 가능한지 정리해봤다. 

# 서론: N+1 문제란?
연관관계에 있는 다른 엔티티를 가져올 때 조회된 데이터의 개수만큼 추가적인 쿼리가 발생하는 것을 N+1 문제라고 한다. 

이렇게만 보면 잘 와닿지 않는다. 그래서 다음과 같은 예제를 가져왔다. 

일단 `Member` 와 `Post` 두 엔티티가 일대다(Member -> Post), 다대일(Post -> Member)로 매핑되어 있다.
- Member 는 여러 개의 Post 를 가질 수 있다. 
- Post 는 여러 개가 존재하며, 각 Post 는 하나의 Member 만 가질 수 있다.

두 엔티티는 `지연로딩` 으로 설정했다. 
이전에 지연로딩에 대해 쓴 글이 있다. 기억이 잘 안난다면 보고 오십쇼 
-> [지연로딩을 통한 성능최적화 + 프록시에 대한 구체적 이해](https://taewoooongheo01.github.io/TaewoooongHeo01/jpa/2024/01/30/lazyAndProxy/)
{% highlight Java%}
@Entity  
@Getter  
@Setter  
public class Member {  
  
	@Id  
	@GeneratedValue  
	private Long id;  
	  
	private String username;  
	  
	@OneToMany(mappedBy = "member")  
	private List<Post> posts = new ArrayList<>(); 

	//연관관계 편의 메서드
	public void addPost(Post post) {  
		posts.add(post);  
		post.setMember(this);  
	}
}
{% endhighlight %}

{% highlight Java%}
@Entity  
@Getter  
@Setter  
public class Post {  
  
	@Id  
	@GeneratedValue  
	private Long id;  
	  
	private String postName;  
	  
	private String content;  
	  
	@ManyToOne(fetch = FetchType.LAZY)  
	@JoinColumn(name = "member_id")  
	private Member member;  
}
{% endhighlight %}

이후 MemberA 를 만들어 Post1, Post2 를 저장하고, 
MemberB 를 만들어 Post3 을 저장했다. 
{% highlight Java%}
Member memberA = new Member();  
memberA.setUsername("memberA");  
  
Member memberB = new Member();  
memberB.setUsername("memberB");  
  
Post post1 = new Post();  
post1.setPostName("post1");  
em.persist(post1);  
  
Post post2 = new Post();  
post2.setPostName("post2");  
em.persist(post2);  
  
Post post3 = new Post();  
post3.setPostName("post3");  
em.persist(post3);  
  
memberA.addPost(post1);  
memberA.addPost(post2);  
em.persist(memberA);  
  
memberB.addPost(post3);  
em.persist(memberB);  

em.flush();  
em.clear();  
  
System.out.println("====================");  

String query = "select m from Member m";  
List<Member> members = em.createQuery(query, Member.class).getResultList();  
  
for (Member m : members) {  
	System.out.println("member="+m.getUsername()+", post="+m.getPosts());  
}
{% endhighlight %}
구현하고자 하는 기능은 각 Member 가 어떤 Post 를 작성했는지 조회하는 기능이다.
위의 예제에서는 Member 만을 찾는 쿼리를 날린 뒤, 해당 Member 에서 직접 Post 찾았다.

결과로 나온 쿼리는 다음과 같다. 
{% highlight SQL%}
====================
Hibernate: 
    /* select
        m 
    from
        Member m 
    join
        m.posts */ select
            member0_.id as id1_0_,
            member0_.username as username2_0_ 
        from
            Member member0_ 
        inner join
            Post posts1_ 
                on member0_.id=posts1_.member_id
Hibernate: 
    select
        posts0_.member_id as member_i4_1_0_,
        posts0_.id as id1_1_0_,
        posts0_.id as id1_1_1_,
        posts0_.content as content2_1_1_,
        posts0_.member_id as member_i4_1_1_,
        posts0_.postName as postname3_1_1_ 
    from
        Post posts0_ 
    where
        posts0_.member_id=?
member=memberA, post=[post1, post2]
member=memberA, post=[post1, post2]
Hibernate: 
    select
        posts0_.member_id as member_i4_1_0_,
        posts0_.id as id1_1_0_,
        posts0_.id as id1_1_1_,
        posts0_.content as content2_1_1_,
        posts0_.member_id as member_i4_1_1_,
        posts0_.postName as postname3_1_1_ 
    from
        Post posts0_ 
    where
        posts0_.member_id=?
member=memberB, post=[post3]
{% endhighlight %}

결과를 보면 Post를 통해 Member를 조회하는 코드에서 추가적인 `SELECT` 쿼리가 발생한다.
{% highlight Java%}
System.out.println("member="+m.getUsername()+", post="+m.getPosts());
{% endhighlight %}

이것을 N+1 문제라고 한다. 한번의 쿼리로 연관 엔티티를 모두 가져오는 것이 아니라, 각 엔티티를 개별적으로 조회함으로써 추가적인 쿼리가 발생한다. 

만약 n 개의 Member 가 각자의 Post 를 모두 조회한다면, n 번의 `SELECT` 쿼리를 통해 Member 를 가져오고, 이후에 각 Member 와 연관된 Post를 추가적으로 `SELECT` 하게 된다. 이는 성능 상 치명적인 문제를 발생시킨다. 

추가적인 쿼리를 발생시키지 않고, Member 를 가져올 때 연관된 Post 를 **단 한번의 쿼리**로 갖고 온다면 이 문제를 해결할 수 있을 것이다.

# EAGER 로 바꾼다면?
그래서 나는 이런 생각이 들었다. 

"나중에 추가적인 쿼리가 발생하는게 문제니까 차라리 `즉시로딩` 을 써서 한번의 쿼리로 한꺼번에 갖고 오면 해결할 수 있지 않을까?"

그래서 `fetch = FetchType.EAGER` 로 설정한 뒤 다시 실행해봤다. 
{% highlight SQL%}
====================
Hibernate: 
    /* select
        m 
    from
        Member m */ select
            member0_.id as id1_0_,
            member0_.username as username2_0_ 
        from
            Member member0_
Hibernate: 
    select
        posts0_.member_id as member_i4_1_0_,
        posts0_.id as id1_1_0_,
        posts0_.id as id1_1_1_,
        posts0_.content as content2_1_1_,
        posts0_.member_id as member_i4_1_1_,
        posts0_.postName as postname3_1_1_ 
    from
        Post posts0_ 
    where
        posts0_.member_id=?
Hibernate: 
    select
        posts0_.member_id as member_i4_1_0_,
        posts0_.id as id1_1_0_,
        posts0_.id as id1_1_1_,
        posts0_.content as content2_1_1_,
        posts0_.member_id as member_i4_1_1_,
        posts0_.postName as postname3_1_1_ 
    from
        Post posts0_ 
    where
        posts0_.member_id=?
member=memberA, post=[post1, post2]
member=memberB, post=[post3]
{% endhighlight %}

결과를 보면 추가적인 `SEELCT` 쿼리가 나가는 것을 확인할 수 있다. -> N+1 해결 못함

이것은 JPA는 JPQL 자체만을 SQL로 해석하기 때문이다.

JPQL은 즉시로딩 그런거 모른다. 만약 쿼리를 "select m from Member m" 라고 짰다면, DB 에서 Member 만을 SELECT 한다. 

하지만, 이때 즉시로딩으로 설정되어 있으므로 JPA에서 추가적인 SELECT 쿼리가 "select m from Member m" 시점에 같이 나간다. 즉, JOIN을 하지 않고 같은 시점에 한번 더 SELECT를 하기 때문에 N+1 문제가 해결되지 않는게 맞다. 

로딩전략을 바꾸는 것 만으로는 N+1 문제를 해결할 수 없음을 알았다.

(추가적으로 N+1 문제와 별개로 즉시로딩은 연관된 엔티티를 모두 로딩하여 성능을 떨어뜨리기  때문에 애초에 쓰면 안된다.)

# JOIN 이라면 어떨까
다시 `fetch = FetchType.LAZY` 으로 지연로딩으로 바꿨다.

그리고 이번엔 `JOIN` 을 사용해봤다. 

로딩전략이 아닌, JPQL 을 쿼리를 직접 바꿨다. 
{% highlight Java%}
"select m from Member m join m.posts"
{% endhighlight %}

실행결과)
{% highlight SQL%}
====================
Hibernate: 
    /* select
        m 
    from
        Member m 
    join
        m.posts */ select
            member0_.id as id1_0_,
            member0_.username as username2_0_ 
        from
            Member member0_ 
        inner join
            Post posts1_ 
                on member0_.id=posts1_.member_id
Hibernate: 
    select
        posts0_.member_id as member_i4_1_0_,
        posts0_.id as id1_1_0_,
        posts0_.id as id1_1_1_,
        posts0_.content as content2_1_1_,
        posts0_.member_id as member_i4_1_1_,
        posts0_.postName as postname3_1_1_ 
    from
        Post posts0_ 
    where
        posts0_.member_id=?
Hibernate: 
    select
        posts0_.member_id as member_i4_1_0_,
        posts0_.id as id1_1_0_,
        posts0_.id as id1_1_1_,
        posts0_.content as content2_1_1_,
        posts0_.member_id as member_i4_1_1_,
        posts0_.postName as postname3_1_1_ 
    from
        Post posts0_ 
    where
        posts0_.member_id=?
member=memberA, post=[post1, post2]
member=memberA, post=[post1, post2]
member=memberB, post=[post3]
{% endhighlight %}
쿼리를 보면 Post 를 조회하는 시점에 `SELECT` 쿼리가 나가는 것을 볼 수 있다. N+1 문제를 해결하지 못했다. 

이것은 `JOIN` 으로 가져오는 연관엔티티는 기본 설정이 "지연로딩" 이기 때문이다. 지연로딩은 프록시 값을 가져오고, 실제 해당 값을 사용할 때 초기화한다. 

따라서 위와 같이 JPQL로 직접 JOIN 을 하더라도 사실상 로딩전략이 지연로딩인 것과 다를게 없다. 

# fetch join 은 가능
N+1 문제를 해결하기 위해 EAGER, LAZY 로 로딩전략도 바꿔보고, 쿼리도 JOIN 을 넣어봤지만 불가능하다는 것을 직접 확인했다. 그렇다면 어떻게 이 문제를 해결할까?

패치 조인(fetch join)으로 가능하다. 패치조인은 한번의 쿼리로 필요로 하는 연관된 엔티티를 한꺼번에 갖고 올 수 있다. 

단 한번의 쿼리로 필요한 엔티티를 모두 가져오기 때문에 불필요한 SELECT 쿼리가 나가지 않는다. 
{% highlight Java%}
"select m from Member m join fetch m.posts"
{% endhighlight %}

실행결과)
{% highlight SQL%}
====================
Hibernate: 
    /* select
        m 
    from
        Member m 
    join
        fetch m.posts */ select
            member0_.id as id1_0_0_,
            posts1_.id as id1_1_1_,
            member0_.username as username2_0_0_,
            posts1_.content as content2_1_1_,
            posts1_.member_id as member_i4_1_1_,
            posts1_.postName as postname3_1_1_,
            posts1_.member_id as member_i4_1_0__,
            posts1_.id as id1_1_0__ 
        from
            Member member0_ 
        inner join
            Post posts1_ 
                on member0_.id=posts1_.member_id
member=memberA, post=[post1, post2]
member=memberA, post=[post1, post2]
member=memberB, post=[post3]
{% endhighlight %}
쿼리를 보면 확실히 줄어든 것을 느낄 수 있다. 

SELECT 절에 Post 엔티티까지 모두 갖고 오는 것이 보인다. 이때 가져온 데이터는 프록시가 아닌, 실제 데이터다. 

이렇게 쿼리를 줄이고 성능최적화를 할 수 있다. 

(N+1 문제를 해결하기 위해선 Fetch Join 뿐 아니라, EntityGraph 어노테이션, Batch Size 등 여러 방법이 있다.)

# 결론
차이점을 정리하면 다음과 같다.

|             | EAGER             | LAZY        | JOIN        | JOIN FETCH             |
| ----------- | ----------------- | ----------- | ----------- | ---------------------- |
| Member 호출 | Member, Post 쿼리 | Member 쿼리 | Member 쿼리 | Member, Post JOIN 쿼리 |
| Post 호출   |                   | Post 쿼리   | Post 쿼리   |                        |
| 총 쿼리     | N+1               | N+1            | N+1            | 1                       |

+패치조인의 더 구체적인 내용은 이후의 포스팅에서 다룰 예정이다(사용법, 주의할 점)

### 참고
자바 ORM 표준 JPA 프로그래밍(기본편) - 김영한

[fetch전략과 fetch join에 관한 오해](https://mighty96.github.io/til/fetch_fetchjoin/)

[N+1 Problem in Hibernate and Spring Data JPA](https://www.baeldung.com/spring-hibernate-n1-problem)


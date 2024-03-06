---
layout: post
title: DispatcherServlet 의 동작원리 하나하나 뜯어보기
category: Spring
excerpt: Dispatcher Servlet 은 기존의 Servlet 에 "dispatch(보내다)" 의 의미가 추가된 형태다. 어쨌든 Dispatcher Servlet 은 Servlet 에 속한다. 다이어그램을 통해 쉽게 확인 가능하다.그렇다면 무엇을 dispatch 하는 것일까? 그것은 DispatcherServlet 에 적용된 패턴들을 살펴보면 알 수 있다. Dispatcher Servlet 은 일반적인 Servlet 과 달리 Front Controller Pattern, Adapter Patten 을 구현하고 있다. Dispatcher Servlet 의 구체적인 동작원리를 살펴보기 전, 위 두 가지 패턴에 대해 알아야 이해가 수월하다.
---

Spring MVC 의 핵심적인 부분 중 하나가 바로 `DispatcherServlet` 이다. 

MVC 프레임워크를 직접 만들 일은 없지만, 동작원리를 이해하는 것이 중요하기에 정리해봤다 <br>+ ~~그냥 보기만하니까 매번 까먹어서 필요성을 느꼈다.~~

관련된 자료는 많지만 복습할 겸, 중간에 내가 헷갈렸던 용어들을 정리하는 식으로 했다.
# Servlet
서블릿이란? 클라이언트와 서버 간의 HTTP 통신을 가능하게 하고, 웹 서버를 통해 웹 페이지를 동적으로 생성할 수 있는 Java 프로그램이다. 

서블릿은 톰캣과 같은 서블릿 컨테이너에 의해 실행되고 관리된다.

서블릿 컨테이너는 `HttpServletRequest`, `HttpServletResponse` 파라미터를 생성해 서블릿 객체에 전달한다. 서블릿 객체는 해당 파라미터들을 이용해 요청과 응답을 처리할 수 있다.

서블릿 객체는 `HttpServlet` 를 상속받아 사용하며, init(), service(), destroy() 등 서블릿의 실행과 생명주기에 관련한 메서드를 제공한다.
# Dispatcher Servlet
Dispatcher Servlet 은 기존의 Servlet 에 "dispatch(보내다)" 의 의미가 추가된 것이다. 어쨌든 Dispatcher Servlet 은 Servlet 에 속한다.

다이어그램을 통해 쉽게 확인 가능하다.
![](https://i.imgur.com/AX3uh1i.png)

그렇다면 무엇을 dispatch 하는 것일까? 그것은 DispatcherServlet 에 적용된 패턴들을 살펴보면 알 수 있다. 

Dispatcher Servlet 은 일반적인 Servlet 과 달리 `Front Controller Pattern`, `Adapter Patten` 을 구현하고 있다. 

Dispatcher Servlet 의 구체적인 동작원리를 살펴보기 전, 위 두 가지 패턴에 대해 알아야 이해가 수월하다.
## Front Controller Pattern
Dispatcher Servlet 은 `Front Controller Pattern` 을 구현하고 있다. Front Controller Pattern 이란 모든 입력을 한 곳에서 받고, 알맞게 분배하는 패턴이다.

Dispatcher Servlet 의 경우, HTTP 요청 url 을 모두 받고, 해당 요청을 처리할 수 있는 핸들러를 찾는다.

Dispatcher Serlvet(Front Controller) 가 없는 경우 요청을 개별적으로 처리해야 하므로, 공통 로직 (보안 검사, 로깅, 요청 전처리 등)을 각 컨트롤러에 반복해서 구현해야 한다. 

이는 하나의 변경사항이 생길 시 모든 코드를 수정해야 하는 것과 같다.
![](https://i.imgur.com/rU25wqO.png)

![](https://i.imgur.com/ula6iQJ.png)

Front Controller 가 없다면
{% highlight Java%}
// LoginServlet.java
public class LoginServlet extends HttpServlet {
   protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
      // 보안 검사
      // 로그인 로직
      // 로깅
      // ...
   }
}

// RegisterServlet.java
public class RegisterServlet extends HttpServlet {
   protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
      // 보안 검사
      // 회원가입 로직
      // 로깅
      // ...
   }
}

// ProfileServlet.java
public class ProfileServlet extends HttpServlet {
   protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
      // 보안 검사
      // 프로필 정보 로딩
      // 로깅
      // ...
   }
}
{% endhighlight %}
만약 보안 정책이 바뀐다면 모든 보안검사 코드를 수정해야 할 것이다.

Front Controller 를 추가해 공통 로직들을 처리한 코드. 
{% highlight Java%}
public class FrontControllerServlet extends HttpServlet {
   protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
      processRequest(request, response);
   }

   protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
      processRequest(request, response);
   }

   private void processRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
      // 공통 보안 검사
      // 공통 로깅

      String path = request.getServletPath();

      switch (path) {
         case "/login":
            new LoginController().handleLogin(request, response);
            break;
         case "/register":
            new RegisterController().handleRegister(request, response);
            break;
         case "/profile":
            new ProfileController().handleProfile(request, response);
            break;
         default:
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            break;
      }
   }
}
{% endhighlight %}
## Adapter Pattern
서로 다른 인터페이스를 가진 두 클래스가 있다고 하자. return, 파라미터 등 차이가 있을 것이다. 이때 사이에서 두 클래스들이 같이 동작할 수 있도록 해주는 것을 어댑터 패턴이라고 한다.

구체적으론 기존 코드를 변경하지 않고 새로운 인터페이스를 기존 시스템에 통합할 수 있게 해준다.

쉬운 예로 우리나라에서 일본 여행을 갈 때 챙겨야 할 것 중 콘센트 어댑터가 있다. 우리나라와 사용하는 전압이 다르기 때문에 콘센트 어댑터를 중간에 사용해야 한다.

한국 플러그
{% highlight Java%}
public class KoreaPlug {  
   
   int voltage;  
   
   public void plugIn() {  
      System.out.println("연결성공");  
   }  
   
   public int getVoltage() {  
      return voltage;  
   }  
   
   public void setVoltage(int voltage) {  
      this.voltage = voltage;  
   }  
}
{% endhighlight %}

일본 소켓. 100V 만 허용한다.
{% highlight Java%}
public interface JapanSocketStandard {  
   boolean connect100V();  
}
{% endhighlight %}

소켓 어댑터. 한국 플러그로 일본 소켓을 사용할 수 있게 해준다.
{% highlight Java%}
public class JapanSocketAdapter implements JapanSocketStandard {  
   
   private KoreaPlug koreaSocket;  
   
   public JapanSocketAdapter(KoreaPlug koreaSocket) {  
      koreaSocket.setVoltage(100);  //전압변환
      this.koreaSocket = koreaSocket;  
   }  
   
   @Override  
   public boolean connect100V() {  
      int voltage = koreaSocket.getVoltage();  
      if (voltage == 100) {  
         koreaSocket.plugIn();  
         return true;  
      }  
      return false;  
   }  
}
{% endhighlight %}

현재 일본이라 `KoreaSocket.plugIn()` 을 바로 실행할 수 없다고 하자. JapanSocketAdapter 를 이용해 스팩에 맞게 바꾸고 사용하여야 한다.
{% highlight Java%}
public class Main {  
   public static void main(String[] args) {  
      JapanSocketAdapter jsa = new JapanSocketAdapter(new KoreaPlug());  
      jsa.connect100V(); //연결성공  
   }  
}
{% endhighlight %}

이렇게 호환되지 않는 인터페이스와 클래스를 서로 연결해주는 패턴을 어답터 패턴이라고 한다.

# 코드로 동작원리 살펴보기
![](https://i.imgur.com/6kwc94s.png)
대략적인 실행 순서다. 

Dispatcher Servlet 은 FrontController Pattern 으로 구현되어 있다. 순서를 살펴보자면...

### doService() 안의 `doDispatch()` 메서드가 실행
`doService()` 메서드는 HttpServlet의 서비스 메서드를 오버라이드하여 구현된다. 그리고 doService() 메서드 안에서 doDispatch() 가 실행된다.

`doDispatch()` 는 Dispatcher Servlet 의 핵심적인 메서드이다. 
![](https://i.imgur.com/6pHdDDj.png)

doDispatch 에서 핸들러에 매핑, 뷰 처리 등 핵심적인 동작이 모두 이루어진다.

### `getHandler()` 가 실행되는 것을 확인할 수 있다 -> 핸들러 매핑 단계
![](https://i.imgur.com/L43gZob.png)

요청에 대한 알맞은 핸들러가 필요하다. 

특정 URI 가 입력되었을 때 해당 URL 를 잡아내는 방법은 여러가지가 있다. 

요즘은 대부분 `@GetMapping` 을 사용하지만, 이전에는 Controller 인터페이스를 구현한 클래스를 빈으로 등록해 URL 을 매핑하는 방법을 사용했다.

이처럼 요청을 처리할 수 있는 핸들러는 여러 종류가 있으며, dispatcher servlet 은 상황에 맞는 핸들러를 갖고 와야 한다.

위 예시처럼 요청을 다양하게 처리하기 위해 사용가능한 핸들러를 `getHandler()` 를 통해 가져온다.

### `getHandlerAdapter()` 가 실행된다. 이때 어댑터 패턴이 구현되어 있다.
가져온 핸들러는 바로 사용할 수 없다. 왜냐하면 핸들러의 종류가 여러가지이기 때문에 `ModelAndView` 라는 통일된 결과로 변환하는 과정이 필요하기 때문이다 (핸들러가 동작하는 방식은 각각의 상황에 맞게 구현되어 있음).

우리는 DispatcherServlet 이라는 하나의 서블릿을 중심으로 실행하고 있기 때문에 리턴결과가 일정해야 한다. 그것을 맞추기 위해 핸들러 어댑터를 사용하는 것이다.

핸들러 마다 각각 사용 가능한 핸들러 어댑터가 모두 다르다. 따라서 각 핸들러에 사용할 수 있는 핸들러 어댑터를 가져와야 한다.
![](https://i.imgur.com/AB30N45.png)

`HandlerAdapter` 의 supports() 메서드를 통해 핸들러 어댑터가 사용할 핸들러를 처리할 수 있는지 확인한다.
![](https://i.imgur.com/6DAVGzO.png)

### `handle()`
핸들러 어댑터가 핸들러를 실행한다. 결과로 `ModelAndView` 를 반환한다.
![](https://i.imgur.com/fa4hR5p.png)

ModelAndView 는 view(view 이름)와 model(데이터)를 가지고 있다.
![](https://i.imgur.com/IXAEYtT.png)

view 와 model 데이터는 이후 뷰를 처리하고 데이터를 보낼 모델을 반환하는 중요한 역할을 한다.
### render, viewResolver
`viewResolver` 는 ModelAndView 를 view 로 반환한다.

viewResolver 또한 핸들러매핑처럼 여러 개가 존재한다. 

view 를 보여주는 방식 또한 다양하고, 각각의 뷰마다 사용할 수 있는 viewResolver 가 모두 다르기 때문이다.

예를 들어 JSP를 처리하는 경우 `InternalResourceViewResolver`, <br>
Thymeleaf 를 처리하는 경우 `ThymeleafViewResolver` 를 사용한다.

이후 반환된 view 에서 `view.render()` 를 통해 뷰가 호출된다.
![](https://i.imgur.com/iwv2wdO.png)

### 정리
![](https://i.imgur.com/iZxZD9F.png)
[출처: Overview of Spring MVC Architecture](https://terasolunaorg.github.io/guideline/5.0.1.RELEASE/en/Overview/SpringMVCOverview.html#overview-of-spring-mvc-processing-sequence)

dispatcher servlet 의 동작원리를 보면 다형성 뿐만 아니라, front controller, adapter pattern 같은 다양한 디자인 패턴들이 실제 적용된 예시를 볼 수 있다. 따라서 공부하는데 많은 도움이 되었다. 

### 참고
스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술 - 김영한

[Servlet (서블릿이란? 그리고, Dispatcher Servlet이란 ? )](https://riimy.tistory.com/87)

https://egovframe.go.kr/wiki/doku.php?id=egovframework:rte:ptl:dispatcherservlet

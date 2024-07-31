
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Servlet 과 SevletContainer](#servlet-과-sevletcontainer)
  - [Tomcat](#tomcat)
  - [서블릿 동작 방식](#서블릿-동작-방식)
  - [서블릿 주요 메서드](#서블릿-주요-메서드)
- [Filter](#filter)
  - [필터 동작 방식](#필터-동작-방식)
  - [필터 주요 메서드](#필터-주요-메서드)
- [인터셉터](#인터셉터)
- [Spring MVC](#spring-mvc)
  - [DispatcherServlet 동작 과정](#dispatcherservlet-동작-과정)
- [질문 모음](#질문-모음)
  - [Q1. Servlet이 무엇인가요? Sevlet 인터페이스는 어떻게 구성되어 있나요?](#q1-servlet이-무엇인가요-sevlet-인터페이스는-어떻게-구성되어-있나요)
  - [Q2. Servlet 계층 구조에 대해 간단히 설명해주세요. 계층 구조를 보면 추상 클래스를 제공하는데 그 이유가 무엇인가요?](#q2-servlet-계층-구조에-대해-간단히-설명해주세요-계층-구조를-보면-추상-클래스를-제공하는데-그-이유가-무엇인가요)
  - [Q3. 서블릿 컨테이너는 무엇이고, C10K 문제는 무엇인가?](#q3-서블릿-컨테이너는-무엇이고-c10k-문제는-무엇인가)
  - [Q4. 필터란 무엇인가요? 주요 메서드는?](#q4-필터란-무엇인가요-주요-메서드는)
  - [Q5. 필터 init 인터페이스 앞에 default 키워드가 붙은 이유는?](#q5-필터-init-인터페이스-앞에-default-키워드가-붙은-이유는)
  - [Q6. 여러 필터가 있을 때, 순서를 제어하는 방법](#q6-여러-필터가-있을-때-순서를-제어하는-방법)
  - [Q7. JSP와 서블릿은 어떻게 다른가?](#q7-jsp와-서블릿은-어떻게-다른가)
  - [Q8. Spring에서 Servlet은 어떻게 구현되어 있나요?](#q8-spring에서-servlet은-어떻게-구현되어-있나요)
  - [Q9. 스프링 필터와 인터셉터의 차이는?](#q9-스프링-필터와-인터셉터의-차이는)
  - [Q10. DispatcherServlet 동작 과정](#q10-dispatcherservlet-동작-과정)

<!-- /code_chunk_output -->


# Servlet 과 SevletContainer

- **서블릿**이란 클라이언트의 요청을 받아 동적 응답을 생성하는 자바 기반 웹 기술이다. **서블릿 컨테이너**란 서블릿을 실행하고 관리하는 환경

```java
public interface Servlet {
  void init(ServletConfig var1) throws ServletException;

  ServletConfig getServletConfig();

  void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;

  String getServletInfo();

  void destroy();
}

public abstract class GenericServlet implements Servlet, ServletConfig, Serializable {
}

public abstract class HttpServlet extends GenericServlet {
}

public class WebServlet extends HttpServlet {
}

public interface ServletRequest {
	Object getAttribute(String var1);

  Enumeration<String> getAttributeNames();

  String getCharacterEncoding();
  
  ...
}

public interface HttpServletRequest extends ServletRequest {
	String getAuthType();

  Cookie[] getCookies();
  
  ...
}

```

## Tomcat

- Apache Tomcat은 웹 애플리케이션(WAS)을 만들고 실행할 수 있게하는 도구. 클라이언트 요청을 처리(Servlet)하고, 동적인 웹 페이지를 생성(JSP)하며, 실시간 통신(Web Socket) 등 기능을 지원
- Tomcat은 서블릿을 실행하고 관리하는 환경을 제공하는 서블릿 컨테이너 역할

## 서블릿 동작 방식

- **클라이언트 요청**: 클라이언트가 서블릿 컨테이너로 HTTP 요청 전송
- **요청 객체 생성**: 서블릿 컨테이너는 `HttpServletRequest`와 `HttpServletResponse` 객체를 생성
- **서블릿 매핑**: 서블릿 컨테이너는 요청 URL을 기반으로 요청을 처리할 서블릿을 검색
- **서블릿 초기화** (필요한 경우): 서블릿이 로드되지 않았다면, 서블릿 컨테이너는 서블릿을 로드하고 초기화
- **요청 처리**: 서블릿 객체는 sevice(doGet, doPost, ...) 메서드를 통해 요청을 처리
- **응답 생성**: 서블릿은 응답을 생성하고, `HttpServletResponse` 객체에 설정
- **응답 전송**: 서블릿 컨테이너는 응답을 클라이언트에게 전송
- **객체 소멸**: 요청 처리가 완료되면 `HttpServletRequest`와 `HttpServletResponse` 객체를 소멸

## 서블릿 주요 메서드

- init(): 서블릿 객체가 초기화될 때 호출
- service(): 클라이언트 요청을 처리
- destroy(): 서블릿 객체가 소멸될 때 호출

# Filter

- 서블릿 요청과 응답을 처리하기 전에 특정 작업을 수행하는 코드

```java
public interface Filter {
  default void init(FilterConfig filterConfig) throws ServletException {
  }

  void doFilter(ServletRequest var1, ServletResponse var2, FilterChain var3) throws IOException, ServletException;

  default void destroy() {
  }
}
```

## 필터 동작 방식

- 요청 → 필터 체인 → 서블릿 → 필터 체인 → 응답

## 필터 주요 메서드

- init(): 필터 객체 초기화
- doFilter(): 필터에서 수행할 작업 수행
- destory(): 필터가 사용한 자원 반환

# 인터셉터

- 인터셉터는 Spring MVC의 요청 처리 과정에서 실행된다. `DispatcherServlet`이 실행된 후, 컨트롤러가 실행되기 전에 요청을 가로채어 전처리 및 후처리 작업을 수행

```java
public interface HandlerInterceptor {
  default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    return true;
  }

  default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {
  }

  default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {
  }
}
```

- 위 메서드로 컨트롤러 **실행 전, 실행 후, 뷰랜더링 후에 실행할 작업을 정의**할 수 있

# Spring MVC

- 사용자 인터페이스와 비즈니스 로직을 분리하고 각각을 독립적으로 유지하여 유연하고 확장할 수 있는 코드 생산을 가능하게 하는 프레임워크

## DispatcherServlet 동작 과정

1. 클라이언트의 요청을 DispatcherServlet에서 수신한다

2. 요청을 처리해 줄 컨트롤러를 HanddlerMapping에게 검색을 요청한다

3. 컨트롤러 정보를 HandlerAdapter에 보내 실행 요청한다.

4. Handler Adapter는 Controller를 통해 비즈니스 로직을 실행한다

5. 실행 결과를 Model에 설정하고 ViewName을 반환한다

6. ViewName으로 View를 찾는 작업을 ViewResolver에게 요청한다

7. 반환된 View에 대한 렌더링 프로세스를 View에 요청한다.

# 질문 모음

## Q1. Servlet이 무엇인가요? Sevlet 인터페이스는 어떻게 구성되어 있나요?

> 서블릿이란 클라이언트의 요청을 받아 동적으로 응답하는 자바 기반 웹 기술. 서블릿 객체를 초기화하고 생성하는 init(), 구체적인 행위를 정의하는 Service(), 서블릿 객체 소멸을 위한 destory() 메서드가 존재

## Q2. Servlet 계층 구조에 대해 간단히 설명해주세요. 계층 구조를 보면 추상 클래스를 제공하는데 그 이유가 무엇인가요?

> 서블릿 인터페이스와 이를 구현하는 추상 클래스인 Generic 서블릿, Generic 서블릿을 상속하는 추상 클래스인 Http 서블릿이 존재한다. 공통 기본 구현을 제공하기 위해 추상 클래스를 사용한다. 하위 클래스에서 재정의해야 할 메서드는 추상 메서드로 정의

## Q3. 서블릿 컨테이너는 무엇이고, C10K 문제는 무엇인가?

> 서블릿 컨테이너란 서블릿을 실행하고 관리하는 환경. C10K문제는 고전적인 문제로 동시사용자 1만명이 접속하는 서버를 구현할 때 발생하는 성능 문제. 서블릿 컨테이너가 요청마다 새로운 쓰레드를 생성하면, 서버의 자원이 빠르게 고갈. 또한, 이렇게 생성된 쓰레드들간의 컨텍스트 스위치로 인해 성능 문제

> 성능 개선 방법으로는 매번 스레드를 새롭게 만들지 않고, 스레드 풀을 만들어 필요한 스레드를 스레드 풀에서 가져와 생성 시간을 줄일 수 있음. 과도한 컨텍스트 스위치를 방지하기 위해 적정 스레드 수 이상으로 만들지 않음. 이벤트 방식을 도입하여 I/O 작업을 하는 동안 쓰레드가 블록킹 되지 않고, 비동기적으로 처리

## Q4. 필터란 무엇인가요? 주요 메서드는?

> 서블릿 요청과 응답을 처리하기 전에 특정 작업을 수행하는 코드. init, doFilter, destory

## Q5. 필터 init 인터페이스 앞에 default 키워드가 붙은 이유는?

> 필터를 구현하는 클래스가 초기화 작업이 필요하지 않을 경우 해당 메서드를 구현하지 않아도 되도록 하기 위함

## Q6. 여러 필터가 있을 때, 순서를 제어하는 방법

> web.xml을 이용해서 순서를 제어하거나 @webFilter와 @order로 순서를 제어할 수 있음

## Q7. JSP와 서블릿은 어떻게 다른가?

> 서블릿은 Java 코드 내부에 HTML 코드가 존재하지만 JSP(Jakarta Server Page)는 HTML 코드 안에 Java 코드가 존재한다. 동적인 웹페이지를 편리하게 생성할 수 있지만 서블릿으로 변환 과정이 필요

## Q8. Spring에서 Servlet은 어떻게 구현되어 있나요?

> Servlet, GenericServlet, HttpServlet 위의 설명 간단히

> Spring에선 HttpServlet을 확장한 DispatcherServlet을 사용. DisPatcherServlet은 요청의 진입점으로 공통 로직 처리. 공통 로직에는 요청  URL을 기반으로 적절한 핸들러를 찾고, 찾은 핸들러를 실행할 수 있는 어댑터를 사용하고, 핸들러 실행 결과를 View로 변환하여 클라이언트에게 응답

## Q9. 스프링 필터와 인터셉터의 차이는?

> 스프링 필터는 서블릿에 대해 동작하지만 인터셉터는 Spring MVC에 대해서만 동작

## Q10. DispatcherServlet 동작 과정

> 1. 클라이언트의 요청을 DispatcherServlet에서 수신
> 2. 요청을 처리해 줄 컨트롤러를 HanddlerMapping에게 검색 요청
> 3. 컨트롤러 정보를 HandlerAdapter에 보내 실행 요청
> 4. Handler Adapter는 Controller를 통해 비즈니스 로직을 실행
> 5. 실행 결과를 Model에 설정하고 ViewName을 반환
> 6. ViewName으로 View를 찾는 작업을 ViewResolver에게 요청
> 7. 반환된 View에 대한 렌더링 프로세스를 View에 요청
> 
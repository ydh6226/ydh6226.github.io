---
title:  "Spring MVC Structure"
excerpt: "스프링 MVC 구조 탐구"

categories:
  - Spring
tags:
  - 스프링
  - mvc
last_modified_at: 2021-09-06T08:06:00-05:00
---
김영한님의 
[스프링 MVC(1)](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-1), 
[스프링 MVC(2)](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-2) 를 보며 정리한 글입니다.

## MVC 전체 구조
![MVC-001](https://user-images.githubusercontent.com/53700256/132339752-498c826b-27f9-48fa-be4f-15c168a8b084.png)

## 요청 흐름
1. 클라이언트가 요청을 보낸다.
2. 클라이언트의 요청을 WAS(Tomcat)이 받고 필터(Filter)를 호출하면서 request, response를 전달한다.
3. 필터체인(Filter Chain)을 따라 요청에 대해 공통된 사항을 적용하고 서블릿(DispatcherServlet)에게 request, response 을 전달한다. 
> 여러 필터들이 순서를 가지고 서로 사슬처럼 연결되있기에 필터체인 이라고 하며 필터체인을 따라 요청을 처리한 다음에 요청이 서블릿에게 넘어간다.
4. HandlerMappins에서 요청을 처리할 수 있는 핸들러를 가져온다.
5. HandlerAdapters에서 핸들러를 호출할 수 있는 어댑터를 가져온다.
6. 등록된 인터셉터들을 루프를 돌며 preHandle을 호출한다. (예외가 발생한 경우 15 과정으로 넘어간다.)
7. requset를 통해 넘어온 argumet를 처리할 수 있는 argumentResolver를 찾는다.
8. argumentResolver가 argument를 핸들러가 받을 수 있도록 변환한다.
> 요청으로 들어온 argumet만큼 argument resolve가 발생한다.
9. 핸들러에서 실제 로직 수행 후 값을 반환한다.
10. 반환값을 처리할 수 있는 ReturnValueHandler를 찾는다.
11. 반환값을 응답형식에 따라 처리한다.
12. ModelAndView 를 반환한다.
> 클라이언트에게 웹페이지를 보여주지 않고 json으로 응답하는 경우는 response에 json값을 넣고 null을 반환한다.
13. 등록된 인터셉터들을 루프를 돌며 postHandle을 호출한다. (예외가 발생한 경우 15 과정으로 넘어간다.)
14. 8 ~ 11 과정 중 예외가 발생했으면 다음 과정으로 넘어가지 않고 이 과정으로 온다. 발생한 예외를 처리할 수 있는 Resolver를 찾고 예외처리를 한다.
15. 등록된 인터셉터들을 루프를 돌며 afterCompletion 호출한다.
> 15번 과정은 예외 발생여부와 상관 없이 항상 수행되며 이전 과정까지 처리가 되지않은 예외들은 afterCompletion메소드의 파라미터로 전달된다.
16. 12 과정에서 반환된 ModelAndView의 viewName으로 View를 생성한다.
17. View를 렌더링해서 response에 넣는다.
> 12 과정에서 null을 반환했다면 16, 17 과정은 수행되지 않는다.

## 디버깅
- DispatcherServlet doDispatch 메소드가 실행된다.

```java
//org.springframework.web.servlet.DispatcherServlet#doDispatch
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  ...
}
```

- 요청을 처리할 수 있는 핸들러를 찾아온다. (요청 URL과 매칭되는 핸들러를 찾아온다.)

```java
//org.springframework.web.servlet.DispatcherServlet#doDispatch
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  ...
  mappedHandler = getHandler(processedRequest);
  ...
}

//org.springframework.web.servlet.DispatcherServlet#getHandler
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
  if (this.handlerMappings != null) {
    for (HandlerMapping mapping : this.handlerMappings) {
      HandlerExecutionChain handler = mapping.getHandler(request);
      if (handler != null) {
        return handler;
      }
    }
  }
  return null;
}
```

- 핸들러 어댑터를 찾아온다.

```java
//org.springframework.web.servlet.DispatcherServlet#doDispatch
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  ...
  HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
  ...
}
```

- 인터셉터목록 루프를 돌며 preHandle을 호출한다. 이때 예외가 발생하면 바로 인터셉터의 afterCompletion을 수행하고 서블릿을 빠져나간다.

```java
//org.springframework.web.servlet.DispatcherServlet#doDispatch
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  ...
  if (!mappedHandler.applyPreHandle(processedRequest, response)) {
    return;
  }
  ...
}

//org.springframework.web.servlet.HandlerExecutionChain#applyPreHandle
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
  for (int i = 0; i < this.interceptorList.size(); i++) {
    HandlerInterceptor interceptor = this.interceptorList.get(i);
    if (!interceptor.preHandle(request, response, this.handler)) {
      triggerAfterCompletion(request, response, null);
      return false;
    }
    this.interceptorIndex = i;
  }
  return true;
}
```

- 실제 핸들러를 호출하기위한 과정이다.

```java
//org.springframework.web.servlet.DispatcherServlet#doDispatch
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  ...
  //이때 json을 반환하는 핸들러라면 mv 는 null이다.
  mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
  ...
}

//org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod#invokeAndHandle
	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
	...
	//디버깅을 쭉 하다보면 결국 여기까지 온다.
	Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
	...
  }
```

- 파라미터를 루프를 돌면서 ArgumentResolver를 찾고 resolve한다.

```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
  ...
  MethodParameter[] parameters = getMethodParameters();
  for (int i = 0; i < parameters.length; i++) {
    MethodParameter parameter = parameters[i];
    parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
    args[i] = findProvidedArgument(parameter, providedArgs);
    if (args[i] != null) {
      continue;
    }
    if (!this.resolvers.supportsParameter(parameter)) {
      throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
    }
    try {
      args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
    }
    ...
}
```

- 실제 핸들러를 호출한다. 

```java
//jdk.internal.reflect.NativeMethodAccessorImpl#invoke0
private static native Object invoke0(Method m, Object obj, Object[] args);
```

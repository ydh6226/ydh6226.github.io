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
![MVC-001](https://user-images.githubusercontent.com/53700256/132216401-5f4bbd06-7bce-4006-a4df-1abbe6ffacd3.jpg)

## 요청 흐름
1. 클라이언트가 요청을 보낸다.
2. 클라이언트의 요청을 WAS(Tomcat)이 받고 필터(Filter)를 호출하면서 request, response를 전달한다.
3. 필터체인(Filter Chain)을 따라 요청에 대해 공통된 사항을 적용하고 Servlet(DispatcherServlet)에게 request, response 을 전달한다. 
> 여러 필터들이 순서를 가지고 서로 사슬처럼 연결되있기에 필터체인 이라고 하며 필터체인이 모두 수행된 다음에 요청이 서블릿에게 넘어간다.
> (스프링 시큐리티를 사용해보면 어마무시한 필터체인을 구경할 수 있다.)
4. HandlerMappins에서 요청을 처리할 수 있는 핸들러를 가져온다.
5. HandlerMappins에서





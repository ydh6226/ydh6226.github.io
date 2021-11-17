---
title:  "Spring REST API 404 응답 커스텀"
excerpt: "SpringBoot를 이용해 Rest API를 작성할 때 404응답을 커스텀하는 방법을 알아보기"

categories:
  - spring
tags:
  - mvc
last_modified_at: 2021-11-17
---


SpringBoot를 이용해 Rest API를 작성할 때 404응답을 커스텀하는 방법을 알아보겠습니다. 관련코드는 [Github](https://github.com/ydh6226/blog-code/tree/master/RESTAPI-440-response-custom)에 있습니다.

잘못된 내용은 댓글로 알려주시면 감사하겠습니다.

# 기존 방식
포스트맨을 이용해서 존재하지 않는 URL에 요청을 해보면 다음과 같은 응답을 하는것을 확인 할 수 있습니다.
```javascript
{
    "timestamp": "2021-07-11T07:20:36.462+00:00",
    "status": 404,
    "error": "Not Found",
    "message": "No message available",
    "path": "/api/v1/bye"
}
```
이 응답은 BasicErrorController의 error함수에 의해 생성되는데요.

_org.springframework.boot.autoconfigure.web.servlet.error.BasicErrorController#error_
![BasicErrorController#error](https://images.velog.io/images/ydh6226/post/c560156a-9584-4947-8fec-7e83c7129a29/image.png)

디버깅을 해보면 body를 구성하는 map에 아래와 같이 이전에 응답 받은 값과 같은 포맷으로 응답을 생성하는것을 확인할 수 있습니다.
![BasicErrorController#error 디버깅](https://images.velog.io/images/ydh6226/post/7a53e885-047a-4497-8d83-867ddd978fc8/image.png)

그렇다면 error함수에 의해 응답을 생성하지 않고 커스텀한 응답을 하려면 어떻게 해야 할까요?

# 4O4응답 커스텀하기
## 설정 추가
우선 프로퍼티에 아래와 같이 설정을 추가합니다. (저는 yml을 사용합니다.)
```yml
spring:
  mvc:
    throw-exception-if-no-handler-found: true
  web:
    resources:
      add-mappings: false
```
>   'spring.web.resources.add-mappings' 와 같은 기능을 하는 'spring.resources.add-mappings' 는 deprecated 되었습니다.


### throw-exception-if-no-handler-found
이 옵션을 ture로 설정하게 되면 dispatcher servlet에서 요청에 대한 핸들러를 찾을 때 요청을 처리할 수 없는 핸들러가 없다면 즉, mappedHandler가 null이라면 NoHandlerFoundException예외를 던집니다.


#### 요청에 대한 핸들러를 찾는 코드
(_org.springframework.web.servlet.DispatcherServlet#doDispatch_)

![DispatcherServlet#doDispatch](https://images.velog.io/images/ydh6226/post/b67eeb34-a339-45b6-8149-c95ae4b8990e/image.png)

- getHandler함수에 의해 핸들러를 찾습니다. 이 함수는 해당되는 핸들러가 없다면 null을 반환합니다.
- 요청을 처리할 수 없는 핸들러가 없다면 noHandlerFound함수를 호출 합니다.

#### 핸들러가 없을 때 어떻게 처리할지 결정하는 함수

_org.springframework.web.servlet.DispatcherServlet#noHandlerFound_
![DispatcherServlet#noHandlerFound](https://images.velog.io/images/ydh6226/post/942d1208-b634-4434-ac1e-4559648f8397/image.png)

- 1276라인에서 throwExceptionIfNoHandlerFound가 true라면 NoHandlerFoundException예외를 던집니다.
- throwExceptionIfNoHandlerFound 값은 프로퍼티파일에서 설정할 수 있습니다.(다른방법으로도 가능합니다.)
- throwExceptionIfNoHandlerFound가 false라면 최종적으로 BasicErrorController에서 응답을 반환합니다.


### add-mappings
'throw-exception-if-no-handler-found' 설정을 이용해 핸들러가 없을 때 예외를 던지도록 설정했는데 'add-mappings'설정은 왜 필요할까요?

우선 'add-mappings'을 true로 설정하고 디버깅을 해보겠습니다.
_org.springframework.web.servlet.DispatcherServlet#getHandler_
![](https://images.velog.io/images/ydh6226/post/2df4bc27-d150-4ae6-a995-0349f68c25d7/image.png)
1256라인에서 handlerMappings에 대해 루프를 돌면서 요청을 처리할 수 있는 핸들러를 찾을 때 매번 handler가 null이여야 할 것 같지만 루프를 돌리다보면 handler가 SimpleUrlHandlerMapping일 때가 있습니다.

존재하지않는 API에 대한 요청을 SimpleUrlHandlerMapping가 받는 이유는 해당 요청을 정적자원에 대한 요청으로 보기 때문인데요.

예를 들면, [GET] localhost:8080/api/v1/test-file 요청을 보낼 때 이 요청을 classpath:/static/api/v1 디렉토리의(기본 설정이라면) 'test-file' 파일에 대한 접근으로 봅니다.

이후 로직은 아래와 같습니다.

- 해당 위치에 파일이 있다면 그 파일을 반환.
- 파일이 없다면 스프링 내부적으로 '/error' URL로 요청을 발생.
- '/error' 에 대한 요청은 RequestMappingHandlerMapping가 받고 최종적으로 BasicErrorController::error함수에 의해 응답 반환.

이처럼 두번의 요청(잘못된 URL 요청, '/error' 요청)이 각각 SimpleUrlHandlerMapping, RequestMappingHandlerMapping에 의해 처리되기 때문에 throwExceptionIfNoHandlerFound를 true로 설정하더라도 NoHandlerFoundException예외가 발생하지 않습니다.

반면에 add-mappings을 false로 설정하면 스프링에서 기본적으로 제공하는 정적자원요청에 대한 매핑을 사용하지 않기 때문에 잘못된 URL로 요청하더라도 SimpleUrlHandlerMapping가 해당 요청을 받지 않고 정상적으로 NoHandlerFoundException을 발생시킵니다.

이제 NoHandlerFoundException이 정상적으로 발생하니 해당 요청을 처리하는 핸들러를 만들어 보겠습니다.

## NoHandlerFoundException 핸들러 생성

NoHandlerFoundException예외를 처리할 수 있는 핸들러를 생성합니다.
```java
@RestControllerAdvice
public class NotFoundHandler {

    @ResponseStatus(HttpStatus.NOT_FOUND)
    @ExceptionHandler(NoHandlerFoundException.class)
    public ErrorResponse handle404(NoHandlerFoundException exception) {
        String message = "존재하지 않는 URL입니다. : " +  exception.getRequestURL();
        return new ErrorResponse(HttpStatus.NOT_FOUND, message);
    }
}
```

공통 응답 클래스
```java
@Getter
@AllArgsConstructor
public class ErrorResponse {
    private int status;
    private String message;

    public ErrorResponse(HttpStatus status, String message) {
        this.status = status.value();
        this.message = message;
    }
}
```

이제 포스트맨을 이용해 잘못된 URL로 요청을 하면 아래와 같이 커스텀한 에러 응답을 반환하는것을 볼 수 있습니다.
```javascript
{
    "status": 404,
    "message": "존재하지 않는 URL입니다. : /api/v1/bye"
}
```

## 정적리소스 경로 추가
지금까지 진행한 방식으로 하게되면 클라이언트는 정적리소스에대한 접근을 할 수 없게됩니다.
하지만 정적리소스접근이 필요할 수도 있기 때문에 해당 요청을 처리할 수 있는 방법을 알아보겠습니다.

정적리소에 대한 접근을 열어두기 위해선 다음과 같은 설정이 필요합니다.
```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/files/**")
                .addResourceLocations("classpath:/static/");
    }
}
```
- addResourceHandler() : '/files' 이하의 모든 요청을 정적리소스에 대한 요청으로 봅니다.
- addResourceLocations() : 정적리소스가 위치한 디렉토리를 추가합니다. '[GET] /files/foo.txt' 요청이라면 'classpath:/static/foo.txt' 를 의미합니다.

실제로 포스트맨으로 '[GET] /files/foo.txt' 로 요청을 보내면 'classpath:/static/foo.txt' 파일내용을 반환합니다.

# 마무리
지금까지 404 REST API응답 커스텀하는 방법이였습니다.


# 참고
https://somoly.tistory.com/131
https://bottom-to-top.tistory.com/39





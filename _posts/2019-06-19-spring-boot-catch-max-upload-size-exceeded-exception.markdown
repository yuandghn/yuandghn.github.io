---
layout:     post
title:      "Catch MaxUploadSizeExceededException in Spring Boot"
subtitle:   ""
date:       2019-06-19 20:05:00
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Spring Boot 
    - MaxUploadSizeExceededException
---
首先，Spring boot 2.0和之前版本的配置是不一样的。2.0是这样
```
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=5MB
```
而2.0之前是
```
spring.http.multipart.max-file-size=5MB
spring.http.multipart.max-request-size=5MB
```
定义一个全局的exception handler

```groovy
@Slf4j
@CompileStatic
@ControllerAdvice
class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(MaxUploadSizeExceededException)
    ResponseEntity<Object> maxUploadSizeExceededExceptionHandler(MaxUploadSizeExceededException e, HandlerMethod handlerMethod ) {
        log.warn("[${getCurrentUserId()}] MaxUploadSizeExceededException occurred, just ignore it")
        return new ResponseEntity(
                [
                      code: code,
                      message: message,
                      extra: extra,
                ],
                HttpStatus.BAD_REQUEST
        )
    }
}
```
测试一下发现没被捕获住，原来是还有个属性需要被设置一下
```
spring.servlet.multipart.resolve-lazily=true
```
不然异常会在请求映射到controller之前被触发。

参考自：https://stackoverflow.com/questions/2689989/how-to-handle-maxuploadsizeexceededexception#answer-54405341



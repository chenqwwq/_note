# HandlerInterceptor

---

[TOC]

---



HandlerInterceptor#afterCompletion 中的异常，可能会被 @ExceptionHandle 拦截并处理，那么方法参数中的异常就会为空。

![image-20211028171245764](assets/image-20211028171245764.png)

上文对于 ex 的解释，并不包括已经被 Exception Resolver 处理过的异常。
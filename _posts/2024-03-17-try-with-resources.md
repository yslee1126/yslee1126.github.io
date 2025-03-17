---
title: try-with-resources 
date: 2025-03-17 19:00:00 +0900
categories: [Java]
math: true
mermaid: true
---

### 개요
- 파일 접근에 대한 자원 해제하는 코드에 가독성이 떨어지는 부분을 발견하여 개선 방법을 알아본다

--- 

### AS-IS 
- 기존 코드들이 file inputstream 을 사용하고 close 하는 코드를 사용하고 있다 

```java
try {
  InputStream in = resource.getInputStream(...);
  
} catch (IO Exception) {
    ...
} finally {
  in.close();
}

```

### TO-BE
- try-with-resources 구문 사용하여 자원을 해제하는 코드를 간결하게 작성한다 

```java
try (InputStream in = resource.getInputStream(...)) {
    ...
} catch (IO Exception) {
    ...
}

```
 
- try 구문이 끝날때 자동으로 할당된 자원을 해제해준다 
- try 안에 io 동작하는 객체 여러개를 세미콜론으로 구분하여 선언할 수도 있다 
- AutoCloseable 인터페이스를 구현한 객체만 사용 가능하다

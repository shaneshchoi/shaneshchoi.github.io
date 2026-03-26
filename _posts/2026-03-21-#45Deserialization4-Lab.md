---
title: Insecure Deserialization 실습(4) - Arbitrary Object Injection in PHP (PortSwigger Academy)
description: 임의 클래스 객체를 주입하여 magic method를 실행시키고 파일 삭제로 이어지는 공격 과정 정리
categories: [Web Security, Deserialization, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Arbitrary object injection in PHP

Lab Link: [Arbitrary object injection in PHP](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-arbitrary-object-injection-in-php)

---

## 핵심 포인트

기존에는 같은 객체에서 값만 수정했다면,  
이번 실습에서는 객체 자체를 바꿔서 코드 실행을 할 예정이다.

---

## 실습 흐름

### 1. 로그인

    wiener : peter

로그인 후 요청을 보면  
session cookie에 serialized object가 들어있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab4-0.png" alt="설명" width="800">
</div>

---

### 2. 소스코드 접근

이번 실습의 목표는 다른 클래스 객체를 주입이다.
따라서 서버에 어떤 클래스가 있는지 알아볼 필요가 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab4-1.png" alt="설명" width="1000">
</div>

Burp에서 **Target → Site map**을 보면  
애플리케이션이 참조하는 파일 목록을 볼 수 있다.

그 중에서 아래 파일을 찾을 수 있는데,

    /libs/CustomTemplate.php

`.php`파일에 라이브러리 경로 (`/libs/`)이기 때문에 클래스 정의 가능성이 높다.

이 외에도 `/lib/`, `/classes/`, `/models/`,`/controllers/`, `/includes/` 등이 있으며,  
간혹 `/index.php`, `/login.php`, `/api/user.php`, `/utils.php` 등 일반 페이지 파일 내부에도 클래스 정의가 있는 경우도 있다.

찾은 `/libs/CustomTemplate.php` 파일을 자세히 살펴보기 위해 Repeater로 보낸다.  
이 상태에서 그냥 요청을 보내게 되면, 실행 결과 (HTTP 200) 만 반환하며 별 다른 응답이 없다.

해당 request를 `GET /libs/CustomTemplate.php~` 이렇게 바꿔준다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab4-2.png" alt="설명" width="800">
</div>

`~` 붙이면 소스코드가 노출되는데, 이렇게 요청 하면 백업 파일/임시 파일이 노출되는 경우가 있다.

서버측 에서는
- `file.php~`
- `file.php.bak`
- `file.php.old`  
등의 파일이 남아있는 경우가 있다. 보통은 에디터 자동 백업, 개발 중 남은 파일, 배포 실수 등으로 남겨진다.

이 요청을 보내면 실행되지 않은 **원본 소스코드**가 그대로 응답에 노출된다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab4-3.png" alt="설명" width="800">
</div>

---

### 3. 취약 클래스 찾기

코드를 보면 이런 부분이 있다.

~~~php
class CustomTemplate {
  private $lock_file_path;

  function __destruct() {
    // Carlos thought this would be  agood idea
    if (file_exists($this->lock_file_path)) {
        unlink($this->lock_file_path);
    }
  }
}
~~~

**Deserialization (Part 2)**에서 살펴본 Magic Method 목록:
- `__construct()` - 객체 생설될 때 실행
- `__wakeup()` - unserialize 될 때 실행
- `__destruct()` - 객체가 사라질 때 실행

이중 `__destruct()`가 존재하는걸 확인할 수 있다.
이 함수는 내부에서 `unlink()`를 실행하며, 경로는 외부에서 들어오는 값이다.

이 클래스를 객체로 만들고, `lock_file_path`값을 조작하면 원하는 파일을 삭제하는것이 가능하다.

### 4. Payload 생성

PHP serialization 형식으로 객체를 만든다.
```
O:14:"CustomTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}
```

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab4-4.png" alt="설명" width="800">
</div>

이 값을 Base64 Encoding을 진행해서 `Cookie`헤더에 Session값으로 넣어준다.

요청이 끝나는 순간 `__destruct()`를 자동으로 실행하여 `/home/carlos/morale.txt`가 삭제되며 Lab이 해결된다.

## 정리

서버에서 쿠키를 역직렬화 시키는데  
1. 어떤 클래스인지 검증이 없으며,  
2. 어떤 값인지 검증이 없기 때문에  

결과적으로 공격자가 원하는 클래스 객체 생성이 가능했다.

---
title: Insecure Deserialization 실습(1) - Modifying serialized objects (PortSwigger Academy)
description: session cookie에 포함된 serialized object를 수정하여 admin 권한을 획득하는 과정 정리
categories: [Web Security, Deserialization, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Modifying serialized objects

Lab Link: [Modifying serialized objects](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-modifying-serialized-objects)

---

## 핵심 포인트

이 lab은 session을 serialized object로 관리한다.

문제는 이 값을 서버가 검증 없이 그대로 `unserialize()` 한다는 점이다.

그래서 cookie 값을 수정하면  
서버 내부 객체 상태를 그대로 바꿀 수 있다.

---

## 실습 과정

### 1. 로그인

주어진 계정으로 로그인한다.

    wiener : peter

로그인 후 `GET` 요청 하나를 보면  
session cookie가 하나 붙는다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab1-0.png" alt="설명" width="800">
</div>

---

### 2. session cookie 확인

Burp에서 요청을 보면 cookie 값이 인코딩되어 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab1-1.png" alt="설명" width="400">
</div>

- URL encoding
- Base64 encoding

Inspector를 이용해 decode 하면 이런 형태가 나온다.

    O:4:"User":2:{s:8:"username";s:7:"wiener";s:5:"admin";b:0;}

여기서 우리가 주목해야 할 값은 `admin` 설정값이다.

    admin → b:0

값이 `0`으로 설정되어 있음으로 사용자는 admin이 아니지만,  
우리가 이 값을 `1`로 변경해서 요청 한 뒤,  
서버 측에서 역직렬화를 진행하면 `admin`유저가 될 수 있다.

---

### 3. 값 수정

Repeater로 요청을 보낸 후  
Inspector에서 cookie를 다시 decode 한다.

그리고 admin 값을 이렇게 바꾼다.

    b:0 → b:1

Apply changes를 누르면 자동으로 다시 인코딩된다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab1-2.png" alt="설명" width="800">
</div>

---

### 4. 요청 재전송

수정된 cookie로 `GET /admin`을 보내면  
admin 권한을 성공적으로 부여 받은걸 확인할 수 있으며,  
admin panel에 엑세스가 되고 `carlos`계정을 삭제하는 링크가 나타난다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab1-3.png" alt="설명" width="500">
</div>

해당 링크를 삭제하는 요청을 보내면서 lab이 완료 된다.

    GET /admin/delete?username=carlos

---

## 가능한 이유

서버 로직을 보면 구조 다음처럼 설정되어 있을 가능성이 높다.

    $user = unserialize($_COOKIE);
    if ($user->admin === true) {
        // admin 기능 허용
    }

문제는 cookie 값을 그대로 신뢰한다는 점이다.

공격자가 serialized object를 수정하면  
서버는 그걸 그대로 객체로 복원한다.

그래서 단순히 `b:0 → b:1` 변경만으로  
권한 상승이 발생한다.


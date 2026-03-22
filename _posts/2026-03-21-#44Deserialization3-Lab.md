---
title: Insecure Deserialization 실습(3) - Using application functionality (PortSwigger Academy)
description: serialized object의 값을 조작해 서버 기능을 악용하여 임의 파일을 삭제하는 과정 정리
categories: [Web Security, Deserialization, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: Using application functionality to exploit insecure deserialization

Lab Link: [Using application functionality to exploit insecure deserialization](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-using-application-functionality-to-exploit-insecure-deserialization)

---

## 핵심 포인트

이 lab은 

**"데이터 조작 → 기능 악용"** 흐름으로 진행된다.

- serialized object 안의 값 (file path)을 조작하고  
- 그 값이 실제 서버 기능 (delete) 에 사용되면서  
- 의도하지 않은 동작이 발생한다  

---

## 실습 과정

### 1. 로그인 및 기능 확인

주어진 계정으로 로그인한다.

    wiener : peter

로그인 후 `/my-account` 페이지로 이동한다.

"My account" 페이지에  
계정 삭제 기능이 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab3-0.png" alt="설명" width="800">
</div>

Burp 에서 확인해 보면 이 기능은 다음 요청을 보낸다.

    POST /my-account/delete

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab3-1.png" alt="설명" width="800">
</div>

---

### 2. session cookie 확인

Burp에서 요청을 확인하고  
session cookie를 decode 한다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab3-2.png" alt="설명" width="400">
</div>

다음과 같은 serialized object가 보인다.

    O:4:"User":...{...s:11:"avatar_link";s:...:"users/wiener/avatar";}

여기서 중요한 필드는 프로필 이미지 경로로, `avatar_link` 값에 해당된다.

---

### 3. 공격 포인트 및 Payload 수정

계정 삭제 기능은 내부적으로  
경로 `users/wiener/avatar`를 사용해서 파일을 삭제한다.

즉, 이 값을 바꾸면  
삭제 대상 파일도 바뀐다.

Repeater에서 cookie를 수정한다.  
다른 값들은 유지하되, `avatar_link`값을 우리의 타겟은 `morale.txt`로 변경 해주자.

    s:11:"avatar_link";s:23:"/home/carlos/morale.txt"

여기서 중요한 점은 경로와 함께 문자열 길이(23)도 같이 수정 해줘야 한다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab3-3.png" alt="설명" width="800">
</div>

---

### 4. 요청 전송 및 결과

수정된 Cookie를 다시 Base64 Encoding 해주고,  
동일한 `POST /my-account/delete` 요청을 보낸다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab3-4.png" alt="설명" width="800">
</div>

내 계정 (wiener)이 삭제됨과 동시에 `/home/carlos/morale.txt`파일도 함께 삭제된다.

이로써 lab이 완료된다.

---

## 가능한 이유

서버 로직은 정상적으로 동작하고 있으나.
```
$user = unserialize($_COOKIE);
```
이 값을 그대로 신뢰한다는 점이 취약점으로 작용된다.
공격자가 cookie를 변조함으로써 사용자 삭제와 동시에 
프로필 이미지 삭제기능을 이용해 다른 파일도 같이 삭제하는게 가능 했다.

---
title: Insecure Deserialization 실습(2) - Modifying serialized data types (PortSwigger Academy)
description: serialized object의 데이터 타입을 조작하여 인증을 우회하고 admin 권한을 획득하는 과정 정리
categories: [Web Security, Deserialization, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: Modifying serialized data types

Lab Link: [Modifying serialized data types](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-modifying-serialized-data-types)

---

## 핵심 포인트

이 lab은 단순히 값을 바꾸는 게 아니라  
**데이터 타입까지 바꿔서 인증을 우회하는 구조**다.

- session이 serialized object로 관리됨  
- 해당 값이 그대로 `unserialize()` 됨  
- PHP의 loose comparison(`==`)이 같이 사용됨  

이 세 가지가 합쳐지면서 인증 우회가 발생한다.

---

## 실습 과정

### 1. 로그인 및 session cookie 확인

주어진 계정으로 로그인한다.

    wiener : peter

로그인 후 아무 `GET` 요청을 확인한다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab2-0.png" alt="설명" width="800">
</div>

Burp에서 요청을 보면 session cookie가 인코딩되어 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab2-1.png" alt="설명" width="400">
</div>

Inspector로 decode 하면 다음과 같은 형태가 나온다.

    O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"..."}

여기서 확인할 수 있는 포인트는 두 가지다.

- username → 현재 계정
- access_token → 인증 관련 값

---

### 2. 공격 방향

이 lab은 "비밀번호 비교"가 아니라  
**token 비교를 우회하는 구조**다.

그리고 핵심은 여기다.

- token은 원래 string
- 비교는 `==` 사용

즉, 타입을 `string`에서 `int`로 바꾸면 우회 가능하다.

Repeater로 요청을 보낸 후 cookie를 수정한다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab2-2.png" alt="설명" width="800">
</div>

> Original Request Base64 Decode

다음과 같이 바꾼다.

    O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:0;}

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab2-3.png" alt="설명" width="800">
</div>

> Modified Request

변경된 부분은 세 가지다.

---

#### 1. username 변경

    "wiener" → "administrator"

그리고 길이도 같이 맞춰야 한다.

    s:6 → s:13

---

#### 2. access_token 값 변경

    "string" → 0

---

#### 3. 타입 변경

    s (string) → i (integer)

그리고 숫자는 따옴표 없이 넣어야 한다.

---

### 3. 요청 전송 및 계정 삭제

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab2-4.png" alt="설명" width="700">
</div>

수정된 cookie를 Base64 인코딩해서 `/admin`에 `GET` 요청을 보내면  
administrator 계정으로 인증이 우회되며 `carlos`계정을 삭제할 수 있는 링크가 생긴다.  

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab2-5.png" alt="설명" width="500">
</div>

`GET /admin/delete?username=carlos`요청을 보내며 lab이 완료된다.

---

## 가능한 이유

서버 로직이 아래와 같이 설정 되어있을 가능성이 있다.

    $user = unserialize($_COOKIE);
    if ($user->access_token == $stored_token) {
        // 로그인 성공
    }

여기서 문제는 두 가지다.

1. user input (cookie 값)을 그대로 deserialize 하는것과  
2. `==` 연산자가 타입을 강제로 맞추는것으로 인해 공격이 가능했다.

우리가 넣은 값:

    access_token = 0 (int)

서버에 저장된 값:

    "randomstring"

비교:

    0 == "randomstring"
    → 0 == 0
    → true

따라서 인증이 우회된다.
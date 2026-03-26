---
title: Authentication 실습(6) - Brute-forcing a stay-logged-in cookie (PortSwigger Academy)
description: Remember-me 쿠키 생성 로직을 분석한 뒤 password wordlist를 이용해 stay-logged-in 쿠키를 brute-force하여 계정에 접근하는 공격
categories: [Web Security, Authentication Vulnerabilities, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Brute-forcing a stay-logged-in cookie

Lab Link: [Brute-forcing a stay-logged-in cookie](https://portswigger.net/web-security/authentication/other-mechanisms/lab-brute-forcing-a-stay-logged-in-cookie)

---

## 핵심 포인트

이 lab의 핵심은 **Remember-me 쿠키 생성 방식이 예측 가능하다는 점**이다.

Stay logged in 옵션을 선택하면 서버는 로그인 상태를 유지하기 위한 쿠키를 생성한다.  
문제는 이 쿠키가 **username과 password hash를 기반으로 만들어진다**는 점이다.

쿠키를 디코딩해 보면 다음과 같은 구조를 확인할 수 있다.

```
base64(username:md5(password))
```

즉 공격자가 password wordlist를 가지고 있다면

1. password를 MD5로 해싱
2. username과 결합
3. Base64 인코딩

과정을 거쳐 **유효한 remember-me 쿠키를 생성할 수 있다.**

이 쿠키를 이용하면 로그인 과정을 거치지 않고도  
**다른 사용자 계정에 접근할 수 있다.**

---

## Wordlist 준비

이번 실습에서는 PortSwigger에서 제공하는 password wordlist를 사용한다.

**Password**:  
[https://portswigger.net/web-security/authentication/auth-lab-passwords](https://portswigger.net/web-security/authentication/auth-lab-passwords)

---

## Solution

### 1. Stay logged in 쿠키 확인

먼저 `wiener:peter` 계정으로 로그인한다.

로그인할 때 **Stay logged in 옵션을 체크**한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab6-0.png" alt="설명" width="600">
</div>

로그인 이후 요청을 확인하면 다음과 같은 쿠키가 설정된 것을 볼 수 있다.

```
Cookie: stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

이 값을 Burp의 Inspector에서 확인한 뒤 **Base64 디코딩**을 해보면 다음과 같은 문자열이 나온다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab6-1.png" alt="설명" width="500">
</div>

```
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

문자열 길이와 형태를 보면 **MD5 hash** 형태라는 것을 쉽게 확인할 수 있다.

> 여기서 MD5 hash인지 확인할 수 있는 방법:
> 1. 길이 (Length) - MD5 hash는 항상 128bit = 16 byte = 32 hex characters
> 2. 문자 구성 (Hexadecimal) - MD5 hash는 일반적으로 **hex encoding**으로 표현된다. ex) 0-9, a-f

정리하자면, `wiener:peter` 계정의 **stay-logged-in cookie**는 이렇게 표현될 수 있다:
```
peter
↓ MD5
51dc30ddc473d43a6011e9ebba6ca770
↓ prefix 추가
wiener:51dc30ddc473d43a6011e9ebba6ca770
↓ Base64
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

---

### 2. Intruder 공격 준비

로그아웃한 뒤 **GET /my-account?id=wiener** 요청을 확인한다.

이 요청을 **Burp Intruder**로 전송한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab6-2.png" alt="설명" width="1000">
</div>

GET 요청에서 `?id=wiener` 파라미터는 제거,  
`stay-logged-in` 쿠키 값을 *payload position*으로 설정,  
이후에 `session`값을 초기화 해주자.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab6-3.png" alt="설명" width="1000">
</div>

---

### 3. Payload processing 설정

Intruder의 Payload 에는 **password wordlist**를 추가 한다.

그 다음 **Payload Processing**에서 아래 규칙을 순서대로 추가한다.

1. Hash → **MD5**
2. Add prefix → `carlos:`
3. Encode → **Base64**

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab6-4.png" alt="설명" width="400">
</div>

이러한 순서를 사용하는 이유는 **서버가 stay-logged-in 쿠키를 생성하는 방식과 동일한 과정을 재현하기 위해서**이다.

앞서 분석한 것처럼 이 애플리케이션의 remember-me 쿠키는 `base64(username:md5(password))` 구조로 만들어진다.
가장 먼저 password를 **MD5로 해싱**, `username:hash` 형태로 문자열 생성, 그리고 마지막으로 해당 문자열을 **Base64 인코딩**하는 순서로 이루어 지기 때문에 Intruder에서도 동일한 순서를 적용해야 한다.

---

### 4. Grep Match 설정

정상적으로 로그인된 상태에서 **My account 페이지**를 보면  
`Update email` 버튼과 `Your username is:`텍스트가 존재한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab6-5.png" alt="설명" width="600">
</div>

따라서 Intruder 설정에서 **Grep Match**를 추가해  
응답에 `Update email`와 `Your username is:`문자열이 포함되는지 확인하도록 설정한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab6-6.png" alt="설명" width="400">
</div>

---

### 5. Carlos 쿠키 brute-force

이 상태에서 공격을 실행하면  
각 password 후보에 대해 **Carlos의 stay-logged-in 쿠키가 생성**된다.

공격 결과를 보면 대부분의 응답에는 `Update email`이나 `Your username is:`라는 문구가 나타나지 않는다.

하지만 **하나의 요청에서 해당 문자열이 포함된 응답이 반환된다.**

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab6-7.png" alt="설명" width="700">
</div>

이 payload가 **Carlos 계정의 유효한 stay-logged-in 쿠키**다.

---

### 6. Carlos 계정 접근

해당 payload로 생성된 쿠키를 사용해 요청을 보내면  
Carlos의 **My account 페이지**에 접근할 수 있다.

이로써 lab이 해결된다.

---

## 정리

이 lab의 핵심은 **Remember-me 쿠키 생성 로직이 예측 가능하다는 점**이다.

쿠키가 취약한 `base64(username:md5(password))` 구조로 만들어졌기 때문에 공격이 가능했다.

이 구조에서는 공격자가

1. password wordlist 준비
2. password MD5 해싱
3. username과 결합
4. Base64 인코딩

과정을 통해 다른 사용자 계정의 쿠키를 직접 생성할 수 있다.  
결과적으로 로그인 과정을 거치지 않고도 계정 접근이 가능해진다.
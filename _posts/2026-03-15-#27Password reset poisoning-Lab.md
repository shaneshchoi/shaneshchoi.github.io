---
title: Authentication 실습(9) - Password reset poisoning via middleware (PortSwigger Academy)
description: X-Forwarded-Host 헤더를 이용해 password reset 링크를 공격자 서버로 유도하여 victim의 reset token을 탈취하는 공격
categories: [Web Security, Authentication Vulnerabilities, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: Password reset poisoning via middleware

Lab Link: [Password reset poisoning via middleware](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-poisoning-via-middleware)

---

## 핵심 포인트

### 주어진 정보

```
Attacker account: wiener:peter
Victim username: carlos
```

Carlos는 이메일로 받은 링크를 **의심 없이 클릭하는 사용자**라고 가정한다.

목표는 **Carlos의 password reset token을 탈취한 뒤 계정에 로그인하는 것**이다.

### 공격 개요

이 lab의 취약점은 **password reset 링크 생성 과정에서 Host 값을 신뢰한다는 점**이다.

password reset 요청이 발생하면 서버는 이메일에 reset 링크를 생성한다.

하지만 이 과정에서 **`X-Forwarded-Host` 헤더 값을 그대로 사용한다.**

따라서 공격자가 이 헤더 값을 조작하면  
reset 링크를 **공격자 서버로 향하도록 만들 수 있다.**

victim이 이메일 링크를 클릭하면

- reset 요청이 공격자 서버로 전달되고
- URL 안에 포함된 **reset token이 노출된다.**

이 token을 이용해 공격자는 victim의 password를 변경할 수 있다.

### Attack flow

```
password reset 요청 분석
→ X-Forwarded-Host 헤더 조작
→ reset 링크를 공격자 서버로 유도
→ victim이 링크 클릭
→ reset token 탈취
→ victim password reset
→ 계정 로그인
```

---

## Solution

### 1. Password reset 요청 확인

로그인 페이지에서 **Forgot your password?** 링크를 클릭한다.

username에 `wiener`를 입력하면  
이메일로 password reset 링크가 전송된다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab9-0.png" alt="설명" width="600">
</div>

---

### 2. Reset 요청 분석

Burp Proxy에서 **POST /forgot-password** 요청을 확인한다.

해당 요청을 **Burp Repeater**로 전송한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab9-1-1.png" alt="설명" width="700">
</div>

요청에 `X-Forwarded-Host:` 헤더를 추가해 전송하면 response 200이 반환된다.

이 경우 password reset 링크 생성 과정에서 **Host 값이 공격자 도메인으로 변경될 수 있다.**

`X-Forwarded-Host` 헤더는 원래 사용자가 요청한 **Host(도메인)** 정보를 전달하기 위해 사용된다.  

주로 **CDN**, **Load balancer**, **reverse proxy** 환경에서 추가된다.

```
AWS ALB
Cloudflare
Nginx
Envoy
Kubernetes ingress
```

이러한 인프라에서는 요청이 여러 프록시를 거쳐 전달되기 때문에  
원래 요청된 Host 정보를 `X-Forwarded-*` 헤더로 전달하는 경우가 많다.

애플리케이션이 이 값을 검증 없이 사용하면  
password reset 링크 생성 과정에서 **공격자 도메인이 포함된 URL이 만들어질 수 있다.**

---

### 3. Reset 링크 조작

따라서 Burp Repeater에서 제공된 **Exploit Server의 URL**과 `X-Forwarded-Host` 헤더를 추가한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab9-3.png" alt="설명" width="1000">
</div>

그리고 요청 body의 username 또한 `wiener`에서 `carlos`로 수정 해주자.

요청을 전송하면 Carlos에게 password reset 메일이 발송된다.

---

### 4. Reset token 탈취

Exploit Server에서 **Access log**를 확인한다.

Carlos가 이메일 링크를 클릭하면  
Exploit server에 아래 요청이 기록된다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab9-5.png" alt="설명" width="600">
</div>

여기서 `temp-forgot-password-token` 값이  
**Carlos의 reset token**이다.

---

### 5. Password reset

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab9-6.png" alt="설명" width="700">
</div>

Burp Repeater에서 URL의 token 값 `GET /forgot-password?temp-forgot-password-token=keir8h2x8owdztv41tqaeshu5d91qki6`을  
**Carlos에게서 탈취한 token으로 교체**한다.

또한, 바꿀 비밀번호를 입력함으로써 비밀번호를 변경 해준다.

---

### 6. Carlos 계정 로그인

설정한 password로 Carlos 계정에 로그인한다.

```
Username: carlos
Password: (new password)
```

로그인 후 **My account 페이지**에 접근하면 lab이 해결된다.

---

## 정리

이 lab의 취약점은 **password reset 링크 생성 과정에서 Host 값을 신뢰한다는 점**이다.

서버는 password reset 요청을 처리하면서  
reset 링크를 생성할 때 `X-Forwarded-Host` 값을 그대로 사용한다.

이 값을 검증하지 않기 때문에 공격자는 요청 헤더를 조작해  
reset 링크의 도메인을 **공격자 서버로 변경할 수 있다.**

```
password reset 요청
→ Host header 조작
→ reset 링크 공격자 서버로 유도
→ victim 클릭
→ reset token 탈취
→ victim password 변경
```

이렇게 노출된 **reset token**을 이용하면  
victim의 password를 재설정하고 계정에 접근할 수 있다.
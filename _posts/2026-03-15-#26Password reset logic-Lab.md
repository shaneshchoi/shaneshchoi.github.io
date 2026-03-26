---
title: Authentication 실습(8) - Password reset broken logic (PortSwigger Academy)
description: Password reset 과정에서 token 검증이 제대로 이루어지지 않는 로직 결함을 이용해 다른 사용자 계정의 비밀번호를 변경하는 공격
categories: [Web Security, Authentication Vulnerabilities, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Password reset broken logic

Lab Link: [Password reset broken logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)

---

## 핵심 포인트

### 주어진 정보

```
Attacker account: wiener:peter
Victim username: carlos
```

목표는 **Carlos 계정의 password를 reset한 뒤 로그인하는 것**이다.

### 공격 개요

이 lab의 취약점은 **password reset 과정에서 token 검증이 제대로 이루어지지 않는다는 점**이다.

일반적인 password reset 과정은 다음과 같다.

1. reset 요청
2. 이메일로 reset 링크 전달
3. reset token 검증
4. password 변경

하지만 이 lab에서는 **password 변경 요청에서 token 검증이 이루어지지 않는다.**

reset 링크 접근에는 token이 필요하지만  
password를 실제로 변경하는 단계에서는 **token 없이도 요청이 처리된다.**

이 구조에서는 요청을 조작해 **다른 사용자 계정의 password를 변경할 수 있다.**

### Attack flow

```
reset 요청 확인
→ reset 요청 분석
→ token 제거
→ username 변경
→ victim password reset
```

---

## Solution

### 1. Password reset 요청 확인

로그인 페이지에서 **Forgot your password?** 링크를 클릭한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab8-0.png" alt="설명" width="600">
</div>

username혹은 email주소를 를 입력하면  
lab에서 제공되는 **Email client**에서 reset 메일을 확인할 수 있다.

메일에는 다음과 같은 reset 링크가 포함되어 있다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab8-1.png" alt="설명" width="600">
</div>

이 링크를 통해 password reset 페이지로 이동한다.

---

### 2. Reset 요청 분석

reset 페이지에서 새로운 password를 입력하고 변경을 진행한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab8-2.png" alt="설명" width="600">
</div>

Burp에서 password reset 요청을 확인하면 아래와 같은 요청이 전송된다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab8-3.png" alt="설명" width="1000">
</div>

헤더와 바디에 `temp-forgot-password-token` 이 중복으로 들어가는걸 볼 수 있으며, 우리가 request보낸 유저 이름과 변경 비밀번호까지 확인이 가능하다.

위에서 확인한 요청을 **Burp Repeater**로 전송한다.

그리고 **URL Parameter**와 **Request Body**에서 token 값을 임의로 랜덤하게 변경해도 **Password reset**이 정상으로 동작하는걸 확인할 수 있다. 

이 말은 서버가 **password 변경 단게에서 token을 검증하지 않는다**는것과 같은 의미다.

---

### 3. 공격 요청 작성

서버가 **token 검증**을 진행하지 않는다는것을 확인 했으니,  
아무 토큰 값을 집어넣고 `username`을 `carlos`로 변경 해준다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab8-4.png" alt="설명" width="1000">
</div>

```
temp-forgot-password-token=randomToken
username=carlos
new-password-1=password
new-password-2=password
```

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab8-5.png" alt="설명" width="600">
</div>

요청을 전송하면 Response 302를 반환,  
**Carlos 계정의 password가 변경된다.**

---

### 4. Carlos 계정 로그인

변경한 password로 Carlos 계정에 로그인한다.

```
Username: carlos
Password: password
```

로그인 후 **My account 페이지**에 접근하면 lab이 해결된다.

---

## 정리

이 lab의 핵심은 **password reset token이 실제 password 변경 단계에서 검증되지 않는다는 점**이다.

reset 링크에 접근할 때는 token이 필요하지만  
password를 변경하는 요청에서는 해당 token이 다시 확인되지 않는다.

즉 reset 페이지에 도달하기만 하면 이후 요청을 조작할 수 있다.

reset 요청 확인  
→ token 제거  
→ username을 victim 계정으로 변경  
→ victim password 재설정  

이 구조에서는 token 없이도 password 변경 요청이 처리되기 때문에  
결과적으로 **다른 사용자 계정의 비밀번호를 임의로 설정할 수 있다.**
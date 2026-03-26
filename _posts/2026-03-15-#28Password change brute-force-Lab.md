---
title: Authentication 실습(10) - Password brute-force via password change (PortSwigger Academy)
description: Password change 기능의 로직 결함을 이용해 victim password를 brute-force하는 공격
categories: [Web Security, Authentication Vulnerabilities, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Password brute-force via password change

Lab Link: [Password brute-force via password change](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-brute-force-via-password-change)

---

## 핵심 포인트

### 주어진 정보

```
Attacker account: wiener:peter
Victim username: carlos
```

목표는 **Carlos 계정의 password를 알아내 로그인하는 것**이다.

---

### 취약점 개념

이 lab은 **password change 기능의 검증 로직 문제**를 이용한다.

password change 요청에는 아래 값이 전달된다.

```
username
current-password
new-password-1
new-password-2
```

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-0.png" alt="설명" width="600">
</div>

입력 값을 조금씩 바꿔보면 서버 반응이 일정하지 않다.  
특히 **current password 검증 결과가 응답 메시지로 드러난다.**

---

### 서버 반응 확인

이번 lab에서는 Password Change 기능에 각각 다른 값을 입력하면서  
서버가 어떤 방식으로 반응하는지 먼저 확인한다.

- Current Password
- New Password
- Confirm new Password

---

#### 1. 틀린 Current Password / 동일한 New Password와 Confirm new Password

예시

```
current-password=test
new-password-1=123
new-password-2=123
```

이 요청을 보내면 에러 메시지가 나타나지 않는다.  
대신 **세션이 종료되고 로그인 페이지로 이동한다.**

다시 로그인을 시도하면  
**Too many attempts. Please try again in 1 minute** 메시지가 나타난다.

---

#### 2. 틀린 Current Password / 다른 값의 New Password와 Confirm new Password

이번에는 current password와 new password를 서로 다른 값으로 입력한다.

예시

```
current-password=test
new-password-1=123
new-password-2=abc
```

이 경우 서버는 **Current password is incorrect** 메시지를 반환한다.

첫 번째 경우와 동일하게 current password가 틀렸지만  
new password 값이 서로 다른 상태에서는 **brute-force protection이 적용되지 않는다.**

---

#### 3. 올바른 Current Password / 다른 값의 New Password와 Confirm new Password

예시

```
current-password=peter
new-password-1=123
new-password-2=abc
```

이 경우 **New passwords do not match** 메시지가 나타난다.

여기서 중요한 점이 드러난다.

current password가 맞는 경우에만  
**New passwords do not match** 메시지가 반환된다.

따라서 carlos 계정을 대상으로 password brute-force 공격을 진행했을 때  
응답 메시지가 **Current password is incorrect**에서  
**New passwords do not match**로 바뀌는 순간  
그 값이 실제 password라는 것을 알 수 있다.

---

### 공격 흐름

```
password change 요청 확인
→ 서버 반응 차이 확인
→ current-password brute-force
→ victim password 식별
→ victim 계정 로그인
```

---

## Solution

### 1. Password change 요청 확인

먼저 `wiener:peter` 계정으로 로그인한 뒤  
Change password 기능을 실행한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-1.png" alt="설명" width="400">
</div>

---

### 2. Repeater에서 요청 테스트

요청을 **Burp Repeater**로 보낸다.  
입력 값을 바꿔가며 앞에서 확인한 서버 반응을 재현한다.

---

#### 1. 틀린 Current Password / 동일한 New Password와 Confirm new Password

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-2.png" alt="설명" width="500">
</div>

> Login 페이지로 redirect 되는 것을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-3.png" alt="설명" width="500">
</div>

> 이후 로그인 시도 시  
> **Please try again in 1 minute(s)** 메시지가 나타난다.  
> brute-force protection이 동작한 것이다.

---

#### 2. 틀린 Current Password / 다른 값의 New Password와 Confirm new Password

1분 뒤 다시 요청을 보낸다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-4.png" alt="설명" width="500">
</div>

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-5.png" alt="설명" width="600">
</div>

> **Current password is incorrect** 메시지가 반환된다.  
> 이번에는 계정 lock이 발생하지 않는다.

---

#### 3. 올바른 Current Password / 다른 값의 New Password와 Confirm new Password

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-6.png" alt="설명" width="600">
</div>

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-7.png" alt="설명" width="600">
</div>

> New password 값이 서로 다르기 때문에  
> **New passwords do not match** 메시지가 반환된다.

이 차이를 통해 **current password 검증 결과가 메시지로 노출된다는 점**을 확인할 수 있다.

---

### 3. Intruder 공격

password change 요청을 **Burp Intruder**로 보낸다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-8.png" alt="설명" width="1000">
</div>

username을 공격 대상인 `carlos`로 변경한다.

`current-password` 파라미터에 **payload**를 설정한다.

payload에는 지금까지 사용해왔던 **password wordlist**를 사용한다.

---

### 4. 공격 결과 확인

Sniper 공격 결과 하나의 요청만 **Response Length가 다른 것**을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-9.png" alt="설명" width="700">
</div>

해당 응답을 보면 **New passwords do not match** 메시지가 나타난다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab10-10.png" alt="설명" width="700">
</div>

또는 **Grep Match**에 해당 문자열을 추가해 필터링할 수도 있다.

payload `cheese`가 **Carlos 계정의 실제 password**다.

---

### 5. Carlos 계정 로그인

찾은 password로 Carlos 계정에 로그인한다.

```
Username: carlos
Password: cheese
```

My account 페이지에 접근하면 lab이 해결된다.

---

## 정리

이 lab은 **password change 기능의 검증 로직 결함**을 보여준다.

입력 값 조합에 따라 서버 반응이 달라진다.  
current password가 맞는 경우에만 `New passwords do not match` 메시지가 나타난다.

이 차이를 이용하면 공격자는 password 후보를 반복적으로 대입하면서  
**victim 계정의 password를 식별할 수 있다.**
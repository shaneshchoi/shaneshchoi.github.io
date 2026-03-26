---
title: Authentication 실습(4) - Username enumeration via account lock (PortSwigger Academy)
description: Account lock 기능의 로직 결함을 이용해 유효한 username을 식별하고 brute-force로 계정을 탈취하는 공격
categories: [Web Security, Authentication Vulnerabilities, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Username enumeration via account lock

Lab Link: [Username enumeration via account lock](https://portswigger.net/web-security/authentication/lab-username-enumeration-via-account-lock)

---

## 핵심 포인트

이 lab의 취약점은 **Account Lock 기능의 로직 결함**이다.

일반적으로 웹 애플리케이션은 brute-force 공격을 방어하기 위해 **로그인 실패 횟수를 제한**하고 일정 횟수 이상 실패할 경우 **계정을 잠그는(Account Lock)** 기능을 사용한다.

하지만 이 lab에서는 특정 username에 대해 여러 번 로그인 시도를 할 경우 **다른 에러 메시지가 반환된다.**

예를 들어 존재하지 않는 username의 경우에는 일반적인 로그인 실패 메시지가 반환되지만, 실제로 존재하는 username에 대해서는 로그인 시도가 누적되면 다음과 같은 메시지가 나타난다.

```
You have made too many incorrect login attempts.
```

이 메시지는 해당 username이 **실제로 존재하는 계정이라는 단서**가 된다.

공격자는 이 차이를 이용해 **유효한 username을 식별(username enumeration)** 할 수 있으며, 이후 password brute-force 공격을 통해 계정에 로그인할 수 있다.

---

### 1. Wordlist 준비

이번 실습에는 PortSwigger Academy에서 제공한 아래 username과 password wordlist를 사용한다:  

**Username**: [https://portswigger.net/web-security/authentication/auth-lab-usernames](https://portswigger.net/web-security/authentication/auth-lab-usernames)  

**Password**: [https://portswigger.net/web-security/authentication/auth-lab-passwords](https://portswigger.net/web-security/authentication/auth-lab-passwords)

---

## Solution

### 1. 로그인 요청 확인

먼저 아무 username과 password로 로그인을 시도한다.

Burp에서 로그인 요청을 확인한 뒤 brute-force 공격을 수행하기 위해  
**POST /login** 요청을 **Burp Intruder**로 전송한다.

---

### 2. Username enumeration

Intruder에서 공격 타입을 **Cluster Bomb**으로 설정한다.

username 파라미터에 payload position을 추가하고, 요청 본문 끝에 빈 payload를 하나 더 추가한다.

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab4-0.png){:width="1200px"}

```
username=§username§&password=password§§
```

이렇게 설정하면 두 번째 payload를 이용해 동일한 username 요청을 여러 번 반복할 수 있다.

비밀번호 payload 앞에 "password"라는 임의 문자열을 추가한 이유는, `§§` payload에 Null payload만 사용하면 password 파라미터 값이 비어 있는 상태로 요청이 생성되기 때문이다.  
이 경우 로그인 요청이 정상적으로 수행되지 않기 때문에, 기본 password 값을 "password"로 설정하여 요청이 항상 유효한 형태로 전송되도록 했다.

---

### 3. Payload 설정

Payload 설정은 다음과 같이 진행한다.

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab4-1.png" alt="설명" width="400">
</div>

**Payload 1 (username)**  
- Payload type: **Simple list**  
- PortSwigger에서 제공한 **username wordlist** 입력  

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab4-2.png" alt="설명" width="400">
</div>

**Payload 2**  
- Payload type: **Null payloads**  
- Payload 개수: **5**

이 설정은 각 username에 대해 **5번씩 로그인 시도**를 발생시키는 효과를 만든다.

예시:
```
Shane : password
Shane : password
Shane : password
Shane : password
Shane : password
Blog : password
Blog : password 
...
```

---

### 4. 공격 결과 분석

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab4-3.png){:width="1200px"}

공격을 실행하면 대부분의 username은 동일한 길이의 응답을 반환한다.

하지만 특정 username에 대해서는 **응답 길이가 더 긴 것을 확인할 수 있다.**

응답 내용을 자세히 확인해보면 다음과 같은 메시지가 포함되어 있다.

```
You have made too many incorrect login attempts.
```

이는 해당 username에 대해 **Account Lock이 발생했다는 의미**이며,  
따라서 이 username: `argentina`가 **실제로 존재하는 계정**이라는 것을 알 수 있다.

---

### 5. Password Brute-force

이제 찾은 username을 이용해 password brute-force 공격을 진행한다.

Burp Intruder에서 새로운 공격을 생성한 뒤 Attack type을 Sniper로 설정한다.

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab4-4.png){:width="1200px"}

username에는 앞 단계에서 확인한 `argentina`를 고정하고, **password 파라미터에 payload position**을 추가한다.

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab4-5.png" alt="설명" width="400">
</div>

이후 PortSwigger에서 제공한 **password wordlist**를 payload로 설정한 뒤 공격을 실행한다.

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab4-6.png){:width="1200px"}

공격 결과를 확인해보면 대부분의 응답은 동일한 Length를 반환하지만, `daniel`을 사용한 요청에서 **응답 Length가 다른 것**을 확인할 수 있다.
이는 해당 요청에서 로그인이 정상적으로 처리되었을 가능성이 높다는 의미이며, 따라서 `daniel`이 올바른 password라는 것을 유추할 수 있다.

보다 명확하게 확인하기 위해서는 이전 Lab에서 사용했던 **Grep - Extract** 기능을 이용해 로그인 실패 메시지를 추출하는 방법도 사용할 수 있다.

---

### 6. 계정 로그인

```
Username: argentina
Password: daniel
```

우리가 찾은 계정으로 로그인 하면서 Lab이 해결된다.

---

## 정리

이 lab의 핵심은 **Account Lock 기능이 username enumeration에 악용될 수 있다는 점**이다.

로그인 실패 횟수를 제한하는 기능은 brute-force 공격을 막기 위한 보안 기능이지만,  
에러 메시지가 다르게 반환될 경우 공격자는 이를 이용해 **유효한 username을 식별할 수 있다.**

유효한 username을 확보한 뒤에는 password brute-force 공격을 수행하여  
결과적으로 **계정 탈취가 가능해진다.**
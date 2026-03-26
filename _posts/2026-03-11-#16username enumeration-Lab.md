---
title: Authentication 실습(1) - Username enumeration via subtly different responses (PortSwigger Academy)
description: 로그인 실패 메시지의 미묘한 차이를 이용해 유효한 username을 식별하고 brute-force로 계정을 탈취하는 공격
categories: [Web Security, Authentication Vulnerabilities, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Username enumeration via subtly different responses

Lab Link: [Username enumeration via subtly different responses](https://portswigger.net/web-security/authentication/lab-username-enumeration-via-subtly-different-responses)

---

## 핵심 포인트

이 lab의 취약점은 **로그인 실패 응답이 완전히 동일하지 않다는 점**이다.

표면적으로는 `"Invalid username or password."` 라는 에러 메시지를 반환한다.

하지만 실제로는 특정 `username`에 대해 응답 메시지에 미묘한 차이가 존재하며, 이 차이를 이용하여 유효한 `username`을 식별할 수 있다.

이후에 마찬가지로 *password wordlist*를 이용해 **brute-force**공격을 수행하면 계정에 로그인할 수 있다.


### 1. Wordlist 준비

이번 실습에는 PortSwigger Academy에서 제공한 아래 username과 password wordlist를 사용한다:  
**Username**: [https://portswigger.net/web-security/authentication/auth-lab-usernames](https://portswigger.net/web-security/authentication/auth-lab-usernames)  
**Password**: [https://portswigger.net/web-security/authentication/auth-lab-passwords](https://portswigger.net/web-security/authentication/auth-lab-passwords)

---

## Solution

### 1. 로그인 요청 확인

먼저 아무 username과 password로 로그인을 시도한다.

Burp 에서 해당 요청을 확인하고 Brute-force 공격을 위해 **Intruder** 로 전송 해주자.

---

### 2. Username enumeration

Burp Intruder에서 username을 *payload position*으로 설정한다.  

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab1-0.png){:width="1200px"}

Payload type은 **Simple list**를 선택, 이후 랩에서 제공된 **username wordlist**를 입력해주자.

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab1-1.png" alt="설명" width="500">
</div>

공격 시작.

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab1-2.png){:width="1200px"}

응답 간에 status code, response, length에서 큰 차이가 없는걸 볼 수 있다.

이때 공격자는 응답으로 돌아오는 로그인 에러 메시지의 일부분을 필터링 할 수 있다.  

**Grep - Extract** 기능을 사용해 응답에서 에러 메시지를 추출해보자.

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab1-4.png" alt="설명" width="500">
</div>

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab1-5.png){:width="1200px"}

기본 응답 메시지인 `invalid username or password.`라는 문구로 필터링 해서, 응답이 다른 항목이 있다면 존재하는 유저일 가능성이 크다.

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab1-4.png" alt="설명" width="400">
</div>

username을 `accounts`로 입력했을때 메시지 맨 뒤에`.`이 없다는걸 확인할 수 있다.

따라서 우리는 `accounts`가 **실제 존재하는 계정** 이라고 유추할 수 있다.

### 3. Password Brute-force

이제 찾은 `accounts`를 username에 고정하고 password를 찾기 위해 brute-force를 진행한다.

마찬가지로 *attack payload*를 password로 변경,  
PortSwigger에서 제공한 password wordlist를 사용하여 공격을 실행해준다.

결과:  
![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab1-7.png){:width="1200px"}

password는 비교적 쉽게 HTTP 응답 필터링 **Redirect (302)** 으로 
`654321` 이라는걸 찾아낼 수 있었다.

이후에    
```
Username: accounts
Password: 654321
```
찾아낸 계정 정보로 로그인 하면 사용자 페이지에 접근할 수 있고 lab이 마무리 된다.

---

## 정리

이 취약점의 핵심은 **로그인 실패 응답이 완전히 동일하지 않다는 점**이다.

겉으로 보기에는 같은 메시지를 반환하는 것처럼 보이지만  
실제로는 **공백, 문장 길이, 철자 등의 차이**가 존재했다.

이 작은 차이만으로도 공격자는 **유효한 username을 식별할 수 있다.**

유효한 username을 확보한 뒤에는 password brute-force 공격을 수행해  
계정 탈취가 가능해진다.
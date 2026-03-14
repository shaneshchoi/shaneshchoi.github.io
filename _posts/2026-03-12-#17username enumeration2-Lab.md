---
title: Authentication 실습(2) - Username enumeration via response (PortSwigger Academy)
description: 로그인 응답 시간 차이를 이용해 유효한 username을 식별하고 brute-force 공격으로 계정을 탈취하는 공격
categories: [Web Security, Authentication Vulnerabilities, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: Username enumeration via response timing 

Lab Link:  
[Username enumeration via response timing](https://portswigger.net/web-security/authentication/lab-username-enumeration-via-response-timing)

---

## 핵심 포인트

이 lab의 취약점은 **응답 시간(response time) 차이**를 이용한 username enumeration이다.

서버는 다음 순서로 인증을 처리한다.

1. username 존재 여부 확인  
2. password 검증  

username이 존재하지 않으면 바로 로그인 실패가 반환된다.  
하지만 username이 존재하면 **password 검증 과정이 추가로 수행된다.**

이 때문에 **유효한 username 요청의 응답 시간이 조금 더 길어진다.**

이번 실습에서는 또 하나의 특징이 있는데, 바로  
로그인 실패가 일정 횟수 이상 발생하면 **IP가 차단된다**는 점이다.

하지만 동시에 서버는 **`X-Forwarded-For` 헤더를 신뢰**한다.  

`X-Forwarded-For` 헤더란, 원래 클라이언트의 IP주소를 전달하기 위해 쓰이는 HTTP 헤더다.  
보통 웹 서비스는 사용자가 서버에 직접 붙지 않고, 중간에 있는 **Reverse Proxy**, **Load balancer**, **CDN**, **WAF**등을 거쳐자며 붙는다.

예를들면 
```text
User → Cloudflare / Nginx / Load Balancer → Web Server
```
이때, 각각 **Client IP**와 **Proxy IP**가 아래와 같이 설정되어있다면:
```text
Client IP: 203.0.113.10
Proxy IP: 10.0.0.5
```
웹 서버측에서 TCP 연결만 봤을때 `10.0.0.5`에서 온 요청처럼 보게 된다.  
이 경우 원래 사용자 IP(Client IP)를 알 수 없기 때문에 프록시가 원래 사용자의 IP를 헤더에 담아서 넘긴다.
```text
X-Forwarded-For: 203.0.113.10
```

따라서 공격 요청에서 `X-Forwarded-For` 값을 임의로 변경할 수 있다면  
서버는 매 요청을 **서로 다른 IP에서 발생한 요청으로 인식**하게 된다.

이렇게 하면 공격 요청을 여러 IP에서 발생한 것처럼 위장할 수 있으며,  
**IP 기반 brute-force 보호를 우회할 수 있다.**

---

## 공격 전략 (Intruder)

이번 실습에서는 Burp Intruder의 **Pitchfork attack**을 사용한다.

목표는 다음 두 가지다.

1. username enumeration (response time 차이 확인)
2. password brute-force

### Pitchfork attack

Pitchfork 공격은 **여러 payload를 동시에 하나씩 매칭**하면서 요청을 보낸다.

예시:
```
payload1 → 1
payload2 → apple

payload1 → 2
payload2 → banana

payload1 → 3
payload2 → orange
```

이번 lab에서는 다음과 같이 사용한다.

| Payload position | 역할 |
|---|---|
| X-Forwarded-For | IP 변경 |
| username | username wordlist |

즉 요청마다 **IP와 username을 동시에 변경**하면서 brute-force 보호를 우회한다.

---

## Solution

### 1. 로그인 요청 확인

먼저 아무 username과 password로 로그인 시도를 한다.  
Burp Proxy에서 해당 `POST /login`페이지를 **Burp Repeater**로 보내서 테스트 해본다.

---

### 2. brute-force 보호 확인

여러 번 로그인 실패를 발생시키면  
서버가 **IP를 차단하는 것**을 확인할 수 있다.

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab2-0.png){:width="1200px"}

이때 요청에 위에서 설명했던 `X-Forwarded-For` 헤더를 추가하면 IP 차단을 우회할 수 있다.

서버가 이 값을 신뢰하는 경우  
공격자는 **임의의 IP를 지정할 수 있다.**

---

### 3. response timing 확인

Repeater에서 여러 테스트를 해보면서 **응답시간 (Response time)**을 확인한다.

**Test 1:**  
유효한 username과 5자리 비밀번호 → 결과: 144 ms
```
username: wiener
password: 12345
```

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab2-1.png" alt="설명" width="400">
</div>

**Test 2:**  
유효한 username과 긴 (50자리) 비밀번호 → 결과: 392 ms
```
username: wiener
password: 1234567890asdqwezxcvfrtgbnhyujpoiuytrewq1232352423
```

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab2-2.png" alt="설명" width="400">
</div>

**Test 3:**  
유효하지 않은 랜덤 username과 긴 (50자리) 비밀번호 → 결과: 141 ms
```
username: shane123
password: 1234567890asdqwezxcvfrtgbnhyujpoiuytrewq1232352423
```

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab2-3.png" alt="설명" width="400">
</div>

위 Test에서 알 수 있듯, uusername이 존재하지 않는 경우는 응답 시간이 140정도로 일정한 반면에,  
**유효한 username**을 입력하면 password 검증이 추가되면서 응답 시간이 더 길어진다.

---

### 4. Username enumeration

이제 Burp Intruder로 공격을 진행한다.
Intruder에서 `X-Forwarded-For`과 `username`을 payload로 설정한다.

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab2-4.png){:width="1200px"}

`password`는 충분히 긴 랜덤 값으로 작성해주고,  
Attack type은 **Pitchfork**를 사용해주자.

Payload 설정 후 공격 실행

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab2-5.png" alt="설명" width="500">
</div>

```
Payload position 1  
→ Numbers  
→ 1 - 100
```

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab2-6.png" alt="설명" width="500">
</div>

```
Payload position 2  
→ Candidate usernames wordlist
```

`X-Forwarded-For`헤더 값이 형식상 진짜 IP처럼 안 보여도, lab 환경에서는 서버가 그냥 값 자체를 식별자로 사용한다. 핵심은 “올바른 공인 IP냐”가 아니라 서버가 그 값을 믿고 다른 클라이언트로 취급하느냐다.

---

### 5. Response time 확인

공격 결과에서 **응답 시간**을 확인하기 위해,  
`Response Received`와 `Response completed` 컬럼을 추가해준다.

이후에 해당 컬럼들로 필터링 해주면, 다른 요청들보다 응답 시간이 조금 더 긴 요청을 찾을 수 있다.

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab2-7.png){:width="1200px"}

결과로 보면 이 `alaska`라는 username이 유효한 계정이라고 추론 할 수 있다.

---

### 6. Password brute-force

이제 username을 방금 찾은 값으로 고정한후에 비밀번호 brute force를 진행한다.

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab2-8.png){:width="1200px"}

**wordlist 선택**

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab2-9.png" alt="설명" width="400">
</div>

이후 **status code**로 필터링 하게되면 로그인 성공을 알리는 **redirect (302)**를 반환 하는 비밀번호 `montana`를 찾을 수 있다.

우리가 찾은 계정으로 로그인 하면서 Lab이 마무리 된다.

```
username: alaska
password: montana
```

## 정리

이 lab은 **response timing 기반 username enumeration** 취약점을 보여준다.

로그인 로직에서 
```
username 확인 → password 검증
```

순서로 처리되는 경우  
유효한 username 요청이 **조금 더 오래 걸리는 특징**이 생긴다.

또한 IP 기반 brute-force 보호가 존재했지만  
`X-Forwarded-For` 헤더를 이용해 **IP spoofing으로 쉽게 우회**할 수 있었다.

결과적으로 공격자는

1. response time을 이용해 username을 식별
2. brute-force로 password를 찾고
3. 이후에 계정으로 로그인 하는것이 가능했다.
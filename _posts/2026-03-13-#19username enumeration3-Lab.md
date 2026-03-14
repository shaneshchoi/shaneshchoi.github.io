---
title: Authentication 실습(3) - Broken Brute-force Protection (PortSwigger Academy)
description: 로그인 실패 카운터 초기화 로직을 이용해 Brute-force 보호를 우회하는 공격 실습
categories: [Web Security, Authentication Vulnerabilities, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: Broken brute-force protection, IP block 

Lab Link:
[Broken Brute-Force Protection](https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block)

## Lab 개요

이번 Lab은 **비밀번호 brute-force 보호 로직의 결함**을 이용하는 문제이다.

웹 애플리케이션은 로그인 실패 횟수를 제한하여 brute-force 공격을 방어하려고 하지만, **로그인 성공 시 실패 카운터가 초기화되는 로직** 때문에 공격자가 이를 우회할 수 있다.

목표는 다음과 같다.

- 피해자 계정의 비밀번호를 brute-force로 찾는다
- 해당 계정으로 로그인한다
- 계정 페이지에 접근하면 Lab이 해결된다

### 제공 정보

```
존재하는 계정 정보:
username: wiener
password: peter

타겟 계정 정보:
username: carlos
password: ?? → 타겟
```

---

## 취약점 분석

로그인 페이지를 확인하면 다음과 같은 동작을 확인할 수 있다.

- 로그인 실패가 **3번 연속 발생하면 IP가 일시적으로 차단**
- 하지만 **정상 로그인에 성공하면 실패 카운터가 초기화됨**

즉 서버의 로직은 다음과 같이 동작한다.

```
로그인 실패 → 실패 횟수 증가
로그인 성공 → 실패 횟수 초기화
```

이 구조는 정상 사용자에게는 편리하지만, 공격자에게는 **brute-force 공격을 지속할 수 있는 우회 방법**이 된다.

공격자는 다음과 같은 패턴으로 로그인 시도를 반복할 수 있다.

1. 피해자 계정으로 여러 번 로그인 시도
2. 차단 기준에 도달하기 전에 자신의 계정으로 로그인
3. 로그인 성공으로 인해 실패 카운터 초기화
4. 다시 피해자 계정 brute-force 진행


![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab3-0.png){:width="1200px"}

확인 결과, 이 실습에서는 로그인 실패 횟수가 3번 연속으로 나올 경우 차단. 
두 번 시도 후에 존재하는 계정으로 한번 로그인 해주는 것 만드로도 실패 카운터를 초기화 할수 있다는 말이다.  
이 방식으로 **IP 차단을 피하면서 brute-force 공격을 계속 수행할 수 있다.**

---

## 공격 전략

Burp Intruder를 사용해 다음과 같은 공격 패턴을 만든다.

```
wiener : peter
carlos : password1
wiener : peter
carlos : password2
wiener : peter
carlos : password3
...
```

파이썬으로 두 번 시도후에 정상계정으로 로그인 하는 script를 쓸 수 있겠으나,  
최대한 BurpSuite으로 주어진 환경에서만 풀어보도록 하겠다.

이 패턴의 핵심은 다음과 같다.

- 공격자의 계정 로그인 → 실패 카운터 초기화
- 피해자 계정 로그인 시도 → brute-force 진행

이 구조를 wordlist 형태로 만들면 **차단 없이 brute-force 공격**이 가능하다.

---

## Exploitation 과정

### 1. 로그인 요청 캡처

Burp Suite를 실행한 상태에서 로그인 페이지에 접근한다.

임의의 username / password를 입력하여 로그인 요청을 발생시키고  
`POST /login` 요청을 **Burp Intruder**로 전송한다.

---

### 2. Intruder 공격 설정

Attack type을 **Pitchfork**로 설정한다.

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab3-1.png){:width="1200px"}

`username`과 `password` 두 곳에 payload를 설정해 준다.

---

### 3. Resource Pool 설정

로그인 요청이 순서대로 실행되도록 설정해야 한다.

Intruder의 **Resource Pool**에서 다음과 같이 설정한다.

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab3-2.png" alt="설명" width="400">
</div>

```
Maximum concurrent requests = 1
```

이 설정은 요청을 **한 번에 하나씩 보내도록 하여 공격 순서를 유지**한다.

---

### 4. Username Payload 설정

Payload position 1에 다음과 같은 리스트를 설정한다.

```
carlos
wiener
carlos
wiener
carlos
wiener
...
```

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab3-3.png" alt="설명" width="400">
</div>

랩 취약점 분석에서 언급했듯이, 이 lab에서는 **세 번 연속**으로 로그인에 실패할 시에 계정이 1분동안 잠기게 된다. 

여기서 주어진 `carlos` 계정으로 중간에 한 번 로그인 해주는것 만으로도 로그인 실패 카운터를 초기화 시켜주기때문에 중간에 유효한 계정으로 로그인 해주는 것이다.

원래라면 두 번 실패 후에 한 번만 로그인 해주면 되나, Burp에서 따로 설정 하는 방법을 몰라서 한 번씩 번갈아가면서 진행해줬다.

---

### 5. Password Payload 설정

Payload position 2에는 이전 Lab들에서 사용했던 **candidate password list**를 사용한다.

단, 각 password 앞에 **유효한 계정의 password**를 삽입한다.

예시

```
123456
peter
password
peter
12345678
peter
qwerty
...
```

<div class="img-left-block">
  <img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab3-4.png" alt="설명" width="400">
</div>

이렇게 하면 요청은 다음과 같은 구조가 된다.

```
carlos : 123456
wiwner : peter
carlos : password
wiener : peter
carlos : 12345678
wiener : peter
carlos : qwerty
...
```

---

### 6. 공격 실행 및 결과 분석

Intruder 공격을 시작한다.

공격이 완료되면 결과를 다음과 같이 분석한다.

1. `Status Code = 200` 응답을 필터링하여 숨긴다  
2. username 기준으로 정렬한다  

![설명](/assets/burp_images/authentication-lab/auth-bruteforce_lab3-5.png){:width="1200px"}

그러면 **carlos 계정에서 발생한 단 하나의 302 응답**을 확인할 수 있다.

이 응답이 나타난 요청의 **Payload 2 값이 정답 비밀번호**이다.

---

## 계정 접근

확인된 비밀번호를 이용해 다음 계정으로 로그인한다.

```
username: carlos
password: qwertyuiop
```

로그인 후 **Account Page에 접근하면 Lab이 해결된다.**

---

## 정리

이 Lab은 **로그인 실패 카운터 초기화 로직의 설계 결함**을 이용하는 문제이다.

웹 애플리케이션은 brute-force 공격을 막기 위해 로그인 시도를 제한했지만, 다음과 같은 이유로 공격을 막지 못했다.

- 로그인 성공 시 실패 카운터 초기화
- IP 기반 제한
- 공격자가 자신의 계정을 이용해 방어 로직 우회 가능

이 사례는 **단순히 brute-force 제한을 구현하는 것만으로는 충분하지 않으며**, 방어 로직이 어떻게 악용될 수 있는지까지 고려해야 한다는 점을 보여준다.
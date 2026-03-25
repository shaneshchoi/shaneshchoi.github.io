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

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab2-0.png" alt="설명" width="700">
</div>

우선은 비밀번호 보호 기능이 어떻게 적용되어있는지 확인하기 위해 의도적으로 틀려준다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab2-1.png" alt="설명" width="700">
</div>

비밀번호를 틀리게 입력하면 3회 이후 계정이 lock되는 것을 확인할 수 있다.

lock이 된 이후에 다른 **username**을 이용해서 로그인 하면 lock이 안되어 있는걸 볼 수 있는데,  
이는 해당 rate limit 이 **username**기준으로 적용되고 있으며 서버에서 관리되고 있음을 알 수 있다.

---

### 2. Race condition 가능성 확인

Burp Repeater에서 요청을 여러 개 복사해서 group으로 묶고 Parallel 요청으로 진행 해보겠다.
만약에 요청이 3회 이상으로 응답을 받으면 Race condition이 존재한다는 가설을 세울 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab2-2.png" alt="설명" width="700">
</div>

Repeater에서 총 15개의 요청을 **Password Request**이름의 그룹으로 묶고 전송을 해본 결과,  
모든 15개의 요청에서 *"Invalid username or password"* 응답을 반환한다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab2-3.png" alt="설명" width="800">
</div>

원래라면 요청 3회 이후에 계정이 잠겨서 "You have made too many incorrect login attempts"를 반환해야 하지만  
15회 모두 정상 요청으로 시도된 것을 확인 했다. → Race condition이 존재한다.

---

### 3. Turbo Intruder 설정

Repeater에서 요청을 선택한 후 'Send to Turbo Intruder'를 클릭해준다.

기본 template에서 `single-packet attack`을 사용 해주고,

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab2-4.png" alt="설명" width="800">
</div>

`username`을 `carlos`로 변경하고 password부분을 *payload*로 설정해주자.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab2-5.png" alt="설명" width="800">
</div>

이후에 템플릿에 있는 코드를 아래와 같이 바꿔준다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab2-6.png" alt="설명" width="800">
</div>

기본 개념 part2 에서 전체적인 코드에 대한 부분은 다뤘기 때문에 바뀐부분에 대해서 설명하도록 하겠다.
PortSwigger에서는 Turbo Intruder의 저자인 James Kettle이 작성한 documentation에 대한 소개글이 있다.  
함께 참조하면 좋을것 같다. [Turbo Intruder: Embracing the billion-request attack](https://portswigger.net/research/turbo-intruder-embracing-the-billion-request-attack)

```python
def queueRequests(target, wordlists):

    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2
                           )
    
    passwords = wordlists.clipboard
    
    for password in passwords:
        engine.queue(target.req, password, gate='race1')
    
    engine.openGate('race1')


def handleResponse(req, interesting):
    table.add(req)
```

`queueRequests`는 요청 준비과 큐에 쌓는 작업을 하며 `handleResponse`는 응답 처리를 한다.  
`.clipboard`는 Turbo Intruder의 내장 wordlist 소스이며, 지금 내 클립보드에 있는 값들을 리스트로 가져온다는 의미다.  

`engine.queue(target.req, password, gate='1')`관련해서,  
Turbo Intruder로 Request에 보낼때 설정된 `password=%s`부분에 대해 payload를 교체 해준다고 보면 되겠다. ex. password=%s → password=qwerty 등등

그리고 중요한 개념인 `gate='1'`는 "아직 보내지 마라" 라는 의미로 해석되는것 같다. 이후에 `openGate('1')`이 나오게 되면 이때 동시에 보내는것으로 설명되어있다. 따라서 동시에 요청을 보내며 rate limit을 우회할 수 있게되는것.

`handleResponse()`는 응답을 테이블에 출력하게 해주는 코드다. 해당 코드를 수정해서 우리가 원하는 결과 값만 출력하게도 만들수 있는데 이 부분은 James Kettle이 작성한 Turbo Intruder 관련 documentation을 참조 하면 자세하게 나와있다.

정리하자면,

1. clipboard에서 password 리스트 가져옴
2. %s placeholder에 하나씩 넣음
3. 요청을 전부 queue에 쌓음
4. openGate()로 동시에 발사
5. 응답 중 성공 (302) 찾음

이런 흐름이라고 이해하면 되겠다.

### 4. 공격 진행

이제 Turbo Intruder에서 공격을 실행 해준다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab2-7.png" alt="설명" width="800">
</div>

공격을 실행하면 응답 중 하나에서 **302 redirect**가 발생하는데, 이 응답이 로그인 성공을 의미한다.  

해당 `password:monkey`를 사용해서 `carlos`계정으로 로그인 해주고

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab2-8.png" alt="설명" width="600">
</div>

*admin panel*에 접근하여 계정을 삭제하면 lab이 해결된다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab2-9.png" alt="설명" width="400">
</div>

---

## 정리

이번 랩에서는 **Race Condition**을 악용하기 위한 **Turbo Intruder** 활용 방법을 실습했다.  
템플릿 구조만 이해하면 다양한 시나리오에서 유용한 테스트 도구로 활용할 수 있다.


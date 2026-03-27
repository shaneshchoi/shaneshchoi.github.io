---
title: Race Conditions (PortSwigger Academy) - 개념 정리 및 응용 (Part 2)
description: hidden multi-step sequence, multi-endpoint race condition, 그리고 single-endpoint race condition을 탐지하는 방법 정리
categories: [Web Security, Race Conditions]
tags: [웹 보안, PortSwigger Academy]
toc: true
---

# Race Condition (Part 2)

Part 1에서는 race condition의 기본 개념과  
limit overrun 형태의 취약점을 살펴봤다.

이번 포스팅에서는 
- Turbo Intruder를 이용한 Single-packet attack
- Hidden multi-step sequence와 sub-state
- Predict, Probe, Prove 방법론
- Multi-endpoint / Single-endpoint race condition
에 대해 다뤄볼 예정이다.

---

## Turbo Intruder와 Single-packet attack

Burp Repeater는 병렬 요청을 빠르게 실험하기엔 좋지만,  
더 복잡한 공격에서는 Turbo Intruder가 더 적합할 수 있다.

PortSwigger 자료에서는 Turbo Intruder가 다음과 같은 상황에 좋다고 설명되어 있다:

1. 여러번 재시도가 필요한 경우
2. 요청 간 타이밍을 조정해야하는 경우
3. 많은 수의 요청을 보내야하는 경우

특히 `HTTP/2`를 서포트하는 대상이라면 Turbo Intruder에서도 **Single-packet attack**을 사용할 수 있다.  
Turbo Intruder는 파이썬 이해능력을 조금은 요구하기때문에 파이썬을 알고 있으면 쉽게 이해 될 듯 하다.

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                            concurrentConnections=1,
                            engine=Engine.BURP2
                            )
    
    # queue 20 requests in gate '1'
    for i in range(20):
        engine.queue(target.req, gate='1')
    
    # send all requests in gate '1' in parallel
    engine.openGate('1')
```

- `Engine.BURP2` → HTTP/2기반으로 요청을 보냄  
- `ConcurrentConnections=1` → 하나의 연결을 사용함
- `gate='1'` → 여러 요청을 같은 gate에 묶음
- `engine.openGate('1')` → gate에 묶인 요청들을 한 번에 전송

Race Condition Lab1 에서 다뤘던 Burp Repeater의 **Send group in parallel**을  
Turbo Intruder에서는 **gate**개념으로 구현하는 셈이다.

`race-single-packet-attack.py` 템플릿을 이용해서 Turbo Intruder 실습을 진행 해보자.

### BurpSuite 예제

Lab : [Bypassing rate limits via race conditions]({% post_url 2026-03-23-#52Race Conditions2-Lab %})

---

## Hidden Multi-step sequences

실제 웹 애플리케이션에서는 하나의 요청이 단순히 한 번의 처리로 끝나지 않는다.  
요청 하나가 내부적으로 **여러 단계를 거치면서**, 짧은 시간동안 다양한 상태를 오가게 된다.

이처럼 요청 처리 과정 중 잠깐 등장했다가 사라지는 중간 상태를 `sub-state`라고 한다.

문제는, 이 짧은 순간의 상태를 공격자가 건드릴 수 있다는 점이다.  
특히 동일한 데이터에 영향을 주는 요청을 동시에 보낼 수 있다면,  
이 Sub-state를 이용해 기존 로직의 타이밍 기반 취약점(race condition)을 유발할 수 있다.

### 예시. MFA 우회 race condition

에를 들어 다음과 같은 로그인 로직이 있다고 가정해보자.

```
session['userid'] = user.userid

if user.mfa_enabled:
    session['enforce_mfa'] = True
    # MFA 코드 생성 및 전송
    # MFA 입력 페이지로 이동
```

이 로직은 하나의 요청 안에서 여러가지 흐름을 가진다:
1. 세션에 사용자 ID 저장
2. MFA 필요 여부 확인
3. MFA 강제 설정
4. 인증 코드 전송

여기서 중요한 포인트는 세션이 생성된 직후부터 MFA가 실제로 enforce되기 전까지의 짧은 구간이다.

로그인은 이미 된 상태지만 MFA는 아직 적용되지 않은 상태로,  
공격자는 이 타이밍에 맞춰 **로그인 요청** + **인증이 필요한 Endpoint**요청을 동시에 보내면 MFA를 우회할 가능성이 생긴다.

따라서 서버는 요청을 순차적으로 안전하게 처리한다고 생각하지만 실제로는 **처리 중간 상태 (Sub-state)가 존재하고,  
이 상태는 동시 요청 (Parallel Request)으로 우회 가능하다. 

### Methodology: Predict, Probe, 그리고 Prove

Hidden multi-step sequence나 race condition을 찾을 때는 무작정 모든요청을 테스트하는 방식으로는 한계가 있다.  
PortSwigger에서는 다음과 같은 접근 방식을 제시한다:

#### 1. Predict - 충돌 가능성 예측

이 단계는, 어디에 **race conditoin**이 생길지 미리 찍어보는 단계다.

모든 endpoint를 테스트하는건 비효율 적이다.  
따라서 먼저 **race condition**이 발생할 수 있는 지점을 좁혀야 한다.

##### 1. 중요한 기능인지 확인하기
- 로그인
- 결제
- 비밀번호 변경
- 권한 변경

##### 2. 같은 데이터로 동시에 건드릴 수 있는지 확인

이게 무슨 말이냐면, 예시로 
```
Request 1 → user A password reset
Request 2 → user B password reset
```
라는 요청을 보내게 되면, 이는 서로 다른 데이터로 출돌이 없는 경우에 해당한다.

반면에,  
```
Request 1 → change email (user A)
Request 2 → change password (user A)
```
라는 요청의 경우, 같은 **user A**에 대한 요청이며 이메일과 비밀번호 변경이 동시에 처리되면 충돌이 생길 수 있다.

다른 예시로는

1. `POST /login` + `GET /admin`: 로그인은 됐는데 인증이 아직 안 됐을 가능성 존재
2. 쿠폰적용 + 쿠폰적용: 쿠폰 중복적용 (이전 Lab 참조)
등이 있다.

#### 2. Probe - 이상 기후 탐지

실제로 요청을 보내면서 이상한 반응이 있는지 확인하는 과정이다.

먼저 요청을 순차적으로 보내서 정상 동작을 먼저 확인해준다.
이후에 같은 요청을 동시에 (Parallel request)보내서 비교 한다.

만약 동시 요청을 보냈을때
- 응답 값이 달라짐
- 상태가 비정상적으로 변경됨
- 이메일, 토큰 등 2차 결과가 달라짐
같은 징후가 있다면 **race condition**의 힌트라고 볼 수 있다.

#### 3. Prove - 취약점 입증

취약점이 존재한다는 가정하에 재현 가능한 exploit을 만들어야 한다.

응답이 다르다고 해서 취약점으로 확정할 수 없기 때문에 반드시 재현 가능해여 하며  
뭘 할수있는지를 실제로 테스트 해보는 단계다. 

실습을 통해서 자세하게 알아보자.

### BurpSuite 예제

Lab : [Multi-endpoint race conditions]({% post_url 2026-03-24-#53Race Conditions3-Lab %})

---

## Single-endpoint race condition

하나의 endpoint에 서로 다른 값을 동시에 보내서 충돌을 유도하는 방법이다.

아래 로직을 예를들어보자.
```
session['reset-user'] = username
session['reset-token'] = token
```

공격자는 이 경우 서버에 Request를 보내 공격자용 Token을 생성한 후에
```
session['reset-user'] = Victim
session['reset-token'] = 공격자 토큰
```
위와 같이 요청하는 형태로
token은 공격자가 받고 대상 user는 victim으로 설정을 하게되면,  
victim 계정 비밀번호 reset이 가능한 race condition 공격이 가능하다.

### 이메일 기능의 취약점

> Emails are often sent in a background thread

PortSwigger에 따르면 서버에서 HTTP Response를 먼저 반환하며  
이메일 전송은 이후에 나중에 비동기로 처리된다고 한다.

**예시. Password Reset 기능 - 구조**
1. 사용자가 `/POST /forgot-password`같은 요청을 보냄
2. 서버가 **reset token**생성, session/state 저장 작업 진행
3. 메일 전송작업을 queue나 background thread에 넘김
4. 서버는 일단 사용자에게 HTTP 응답을 먼저 돌려줌
5. 그 뒤에 별도 thread가 실제 이메일을 전송함

이 구조에서는 요청 처리와 실제 메일 발송 사이에 시간차가 발생하며,  
이 구간이 **race window를 넓히는 요소**가 된다.

### 이메일 기반 기능이 취약한 이유

#### 1. 외부 시스템의 개입
이메일은 SMTP 서버, 메일 provider, queue worker 같은 외부 컴포넌트를 거치는 경우가 많다.  
이로 인해 처리 지연이 발생하고, 비동기 구조가 자연스럽게 사용된다.

#### 2. 빠른 응답 처리 (인척하기)
사용자는 버튼 클릭 후 오래 기다리는걸 싫어한다.  
그래서 서버는 보통 "메일 보냈습니다" 같은 응답을 먼저 던져주고 실제 메일은 뒤에서 보내는 경우가 존재한다.

이 구조가 race condition 발생 가능성을 높인다.

### Test 포인트

이메일 관련 기능 중 race condition 확인해볼만한 포인트들이다:
- 비밀번호 재설정
- 이메일 주소 변경 확인
- 회원가입 인증 메일 재전송
- 초대 링크 발송
- 2FA 해제/복구 메일
- Magic link 로그인

공통점은 전부 **토큰 생성** + **사용자 식별** + **이메일 발송**이 묶여 있다는 점이다.

예제와 함께 알아보자.

### BurpSuite 예제

Lab : [Single-endpoint race conditions]({% post_url 2026-03-25-#54Race Conditions4-Lab %})

---

## Time-Sensitive attack (타이밍 기반 공격)

이전에 살펴봤던 race condition은 state 충돌로 발생하는 반면에  
**타이밍 기반 공격**은 같은 타이밍에 실행되게 해서 같은 결과를 만들어내는 공격이다.

### Scenario 

유저 A와 유저 B가 동시에 Password Reset 요청을 보냄. 그리고  
만약에 개발자가 토큰을 만들 때 다른 안전장치 없이 Timestamp를 기준으로 만든다고 가정해보자.

Timestamp라는건 "지금 이 순간의 시간을 숫자로 표현한 값" 이다.  
말인 즉슨, **같은 시간에 만들어진 Token은 같은 값을 가지게 된다**는 취약점이 있다.

두 요청이 동일한 시점 (예: 12:00:00)에 처리되면:   
**유저 A 토큰 = 123456**
**유저 B 토큰 = 123456**
와 같이 동일한 token이 생성될 수 있다.

만약에 비밀번호 초기화 링크에 해당 토큰과 username이 포함되어있다고 하면
```
www.vulnerable-website.com/reset/userId=userA&token=123456
www.vulnerable-website.com/reset/userId=userB&token=123456
```
와 같은 링크가 각각 하나씩 유저에게 보내지는 원리다.

공격자는 자신의 tokne을 알고 있기 때문에, `userId`만 victim id로 바꿔 요청하면 계정 탈취가 가능해진다.  
실습과 함께 알아보자.

### BurpSuite 예제

Lab : [Exploiting time-sensitive vulnerabilities]({% post_url 2026-03-25-#55Race Conditions5-Lab %})

---

## 방어 방법

race condition의 방어 본질은

**같은 자원(state)을 여러 요청이 동시에 건드려도, 시스템이 이상한 상태에 빠지지 않는 것**이다.

크게는 두 가지로 귀결 될 수 있다:
1. 예측 가능한 값 쓰지 않기
2. 상태 변경을 끼어들 수 없게 만들기

---

### 1. 랜덤 생성 (가장 기본)

Time-sensitive attack 방어 쪽에서 가장 기본적이다.

- 운영체제 수준의 **cryptographically secure random**사용 한다.
- 토큰은 충분히 길고, 충돌 가능성이 사실상 없게 생성 한다.
- 필요하면 만료시간은 토큰과 별도로 저장하거나 검증 한다.

### 2. 한 번에 처리하기

예를 들어 실제 주문 처리 로직을 생각해봤을때
재고확인 → 결제 금액 확인 → 주문 확정 → 재고 차감

위 프로세스를 따로 진행하면, 그 사이에 다른 요청이 끼어들 수 있다.  
따라서 검증은 통화했는데, 확정 시점에는 조건이 달라진 상태가 발생한다.

이걸 막기 위해서는 **검증**와 **변경**을 하나의 묶음으로 처리해야 한다.

- Transaction
- Row-level lock
- Compare-and-swap
- Optimistic locking

같은 매커니즘을 사용하면, 확인한 조건이 실제 변경 시점까지 그대로 유지되도록 보장할 수 있다.

### 3. Session도 안전하지 않다

Session은 사용자 간에는 분리되지만,  
같은 사용자의 요청 간에는 공유된다.

즉, 동시에 여러 요청이 들어오면  
같은 session 값을 서로 덮어쓸 수 있다.

비유하자면,  
화이트보드 하나에 두 사람이 동시에 값을 적는 상황이라고 가정하자.

- A가 "user = attacker" 작성  
- B가 "user = victim" 덮어씀  
- A가 "token = AAAA" 작성  

결과:
```
user = victim
token = AAAA
```

서로 다른 요청의 값이 섞이기때문에 안전하지 않다.

### 4. DB 제약조건 활용

Application에서 여러가지 로직 문제나 코드 취약점이 발생할 수 있다. 
마지막 보루로 DB 제약조건을 설정해주면 방어를 더 견고하게 만들 수 있다.

예를 들어
- password reset token을 unique하게 만듦
- 같은 coupon은 한번만 사용 가능
- 같은 초대 링크는 한 번만 소비 가능
- 동일 계정에 동시에 하나의 password reset만 허용

이런 제약은 Application 로직이 실수해도 DB에서 차단시켜준다.

### 5. Server-side state 최소화

서버가 매 요청마다 중간 상태 (sub-state)를 들고 있으면,  
그 상태가 race의 표적이 된다. 그래서 어떤 시스템은 아예 서버 상태를 최소화 하려고 한다.  

예를들어 클라이언트가 서명된 상태값을 들고 오게 하는 방식이다.  
JWT같은게 대표적인데, 이 방식의 장점은:

- 서버 메모리/session 의존도 감소
- 상태 공유 문제 감소
- 수평 확장도 쉬움
등이 있다. 

하지만 이는 race condition을 줄여줄 수는 있어도, 보안을 자동으로 해결해주는 만능은 아니다.
JWT 서명키 관리 실패 가능, replay문제등이 발생할 수 있다. 즉, 서버상태를 없애면 무조건 안전한게 아니라  
다른 종류의 보안 문제와 운영 복잡성으로 바뀐다고 봐야한다.

---

출처: [PortSwigger Academy](https://portswigger.net/web-security/race-conditions)
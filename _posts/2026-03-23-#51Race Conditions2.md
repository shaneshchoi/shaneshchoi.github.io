---
title: Race Conditions (PortSwigger Academy) - 개념 정리 및 응용 (Part 2)
description: hidden multi-step sequence, multi-endpoint race condition, 그리고 single-endpoint race condition을 탐지하는 방법 정리
categories: [Web Security, Race Conditions]
tags: [웹 보안, Race Conditions]
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

출처: [PortSwigger Academy](https://portswigger.net/web-security/race-conditions)
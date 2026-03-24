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

출처: [PortSwigger Academy](https://portswigger.net/web-security/race-conditions)
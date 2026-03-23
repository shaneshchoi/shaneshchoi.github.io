---
title: Race Conditions (PortSwigger Academy) - 개념 정리 (Part 1)
description: Race condition의 개념과 공격 원리, TOCTOU 취약점 이해
categories: [Web Security, Race Conditions]
tags: [웹 보안, Race Conditions]
toc: true
---

# Race Condition

Race condition은 웹 애플리케이션에서 발생하는 대표적인 **로직 취약점**이다.

서버가 여러 요청을 동시에 처리하는 과정에서  
같은 데이터에 대한 상태 검증과 변경이 겹치면서  
의도하지 않은 결과가 발생한다.

---

## 핵심 개념

Race condition의 본질은 다음과 같다.

1. 서버는 요청을 순차적으로 처리하지 않는다
2. 여러 요청이 동시에 실행될 수 있다
3. 검증(check)과 처리(use) 사이에 시간 차이가 존재한다

이 틈을 이용하면 공격이 가능해진다.

---

## Race Window

Race window는  
두 요청이 충돌할 수 있는 **매우 짧은 시간 구간**이다.

할인 코드 적용 로직을 예로 들어보자.  
쇼핑몰에 적용할 수 있는 할인코드 `ShaneBlog`라는 코드를 사용자가 입력 했을때 
다음과 같은 순서로 검증과 처리가 진행된다.  

1. 코드 사용 여부 확인 - 서버가 쿠폰 사용여부를 체크함
2. 할인 적용 - 미사용시 할인 적용
3. 사용 완료 상태로 DB 업데이트 - DB에 "사용 완료" 기록

위 과정에서 **쿠폰 사용 검증과 DB 업데이트 사이**에 짧은 시간 간격이 존재한다.

이때 여러 요청을 동시에 보내면  
```
Request A → 쿠폰 적용
Request B → 쿠폰 적용
```
둘 다 할인 적용될 수도 있다.

---

## TOCTOU (Time-of-Check to Time-of-Use)취약점

**TOCTOU(Time-of-Check to Time-of-Use)**은 Race Condition의 대표적인 형태다.  
TOCTOU는 검증 시점(check)과 실제 사용 시점(use) 사이의 시간차를 악용하는 취약점이다.

이 취약점은 여러 요청을 동시에 보냈을때 서버 순서가 꼬여서 충돌을 유도하게끔 만든다.

대표적인 공격 사례로는  
- 1회 제한 기능 (쿠폰, 기프트 카드)
- 금액 관련 기능 (송금, 출금)
- 인증/권한 체크 (Rate limit 우회)
등이 있다.

---

## 실무 방법

이론적으로는 요청을 동시에 보내면 가능한 공격이지만 앞서 소개한 **Race Window**는 **ms**단위로 매우 짧으며,  
- 네트워크 지연
- 서버 처리 순서
- 내부 latency
등을 고려했을때 일반적인 방법으로는 재현이 어렵다.

### Burp를 이용한 해결 방법 

PortSwigger에서는 요청을 최대한 동시에 보내는것을 강조하며  
BurpSuite를 이용해서 위 문제점을 해소할 수 있는 방법을 소개한다. 

1. Parallel Request - 병렬 요청 기능을 통해 여러 요청을 동시에 전송
2. Single Packet Attack - HTTP/2 환경에서 여러 요청을 하나의 TCP 패킷으로 전송하여 네트워크 지연 영향을 제거 

#### 1. Parallel Request (HTTP/1)
`Parallel Request`는 요청을 여러 개 준비한 뒤, 가능한 한 동시에 **끝부분**을 보내도록 조정하는 것이다.  
"끝부분"을 동시에 보내도록 조정하는 이유: HTTP 요청은 서버가 완전한 요청을 다 받아야 처리 시작하는 경우가 많기 때문.  
이 요청은 `Classic Synchronization` 계열의 기법이다.

요청 헤더와 바디 일부만 먼저 보내놓고  
마지막 바이트를 여러 요청에 대해 거의 같은 순간에 보내면  
서버 입장에서는 이 요청들이 거의 동시에 완성된다.

이 경우,  
네트워크 상황이 완벽하지 않아도, **마지막 Byte를 보내는 타이밍**을 최대한 맞추면 실제 처리 시작 시점 차이를 줄일 수 있기 때문에 효과가 있다.  
다만, HTTP/1에서는 보통 여러 연결이 필요하고, 각 연결마다 미세한 차이가 남을 수 있다. 그래서 어느 정도 *jitter*는 여전히 존재한다.  

> *jitter*란 요청이 같이 출발했는데도 서로 다른 시간에 도착하거나 처리되는 현상

#### 2. Single-Packet Attack (HTTP/2)

HTTP/2의 경우,  
HTTP/1처럼 요청마다 연결이 따로따로 필요한 구조가 아니라, 하나의 TCP 연결 위에서 여러 요청을 동시에 다룰 수 있다.  
이 특성을 이용해서 여러개의 HTTP/2 요청을 조립한 뒤, 완료 시점을 하나의 TCP packet 안에 몰아넣는식으로 요청을 한다고 한다.

일반적으로 요청 여러개를 보내게 되면, 각 요청이 네트워크를 따로 타고 가면서 미세한 지연 차이가 생기지만,  
`Single-Packet Attack`에서는 걸정적인 마지막 부분이 하나의 **TCP Packet**에 함께 담겨 간다.

때문에 같은 경로를 이동하고, 같은 순간에 도착하고, 같은 packet안에서 전달되므로 요청별 도착 시간차이가 극도로 줄어들게 된다.

Race Condition에서 실패 원인의 상당수가 "서버가 취약하지 않아서"가 아니라 단순히 요청 타이밍이 안 맞아서 생기는 경우가 많은데, Single-Packet Attack은 이 타이밍 문제를 강하게 보강해주는 방법으로 사용된다.

---

## 그럼에도 실패할수 있는 요인

위 방법들로 **Network Jitter**는 줄일수 있지만,  
**Internal Latency**, 즉 **Server-side Jitter**까지 완전히 없애는 건 아니다.  

서버 내부에 아래와 같은 요소들이 존재할 수 있다:
- 스레드 스케줄링 차이
- DB 락 대기
- 캐시 접근 타이밍
- 애플리케이션 로직 분기
- CPU 점유 상태

네트워크는 맞췄어도 서버 내부에서는 각각의 요청이 동시에 도착하지 못할 확률이 있다.  
때문에 동시에 요청을 2개만 보내기보다,  
가능한 한 많은 수의 요청(20, 30개)을 한번에 보내는 것이 적어도 일부 요청끼리는 같은 **race window**에 겹칠 확률이 더 높다고 설명된다.

다음 Part에서는 실제 PortSwigger Lab을 통해  
race condition을 어떻게 exploit하는지 단계별로 정리할 예정이다.

---

출처: [PortSwigger Academy](https://portswigger.net/web-security/race-conditions)
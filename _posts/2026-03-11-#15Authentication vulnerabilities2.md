---
title: Authentication Vulnerabilities - 비밀번호 기반 로그인
description: Password 기반 로그인에서 발생할 수 있는 인증 취약점과 공격 기법
categories: [Web Security, Authentication Vulnerabilities]
tags: [웹 보안, Authentication Vulnerabilities]
toc: true
---

# Authentication Vulnerabilities - Password-based Login

웹 애플리케이션에서 가장 일반적으로 사용되는 인증 방식은 **password-based login**이다. 사용자는 계정을 생성하거나 관리자로부터 계정을 부여받고, **username과 password**를 이용해 로그인한다.

이 방식에서는 **사용자가 비밀 패스워드를 알고 있다는 사실 자체가 곧 사용자 신원 증명**이 된다. 따라서 공격자가 다른 사용자의 로그인 정보를 **획득하거나 추측할 수 있다면 인증 체계는 쉽게 무너질 수 있다.**

대표적인 공격 방식은 다음과 같다.

- Brute-force attacks  
- Username enumeration  
- HTTP Basic Authentication 관련 취약점  

이번 글에서는 **Brute-force 공격과 Username enumeration**을 중심으로 살펴본다.

---

## Brute-force 공격이란?

**Brute-force attack**은 공격자가 **수많은 username과 password 조합을 반복적으로 시도하여 유효한 인증 정보를 찾는 공격 방식**이다.

일반적으로 다음과 같은 특징을 가진다.

- 자동화된 도구 사용 (Burp Intruder 등)
- username / password wordlist 활용
- 매우 빠른 속도로 대량 로그인 시도

이 공격은 단순히 무작위 추측을 하는 것이 아니라 **현실적인 패턴과 정보를 활용하여 효율을 높인다.**

예를 들어:

- 이메일 형식 username
- 자주 사용되는 패스워드 목록
- 회사 내부 사용자 네이밍 규칙

웹사이트가 **충분한 brute-force 방어 정책을 적용하지 않은 경우** password-based authentication은 매우 취약해질 수 있다.

---

## Brute-forcing usernames

Username은 종종 **예측 가능한 형태로 생성되기 때문에 공격자가 쉽게 추측할 수 있다.**

대표적인 예시는 다음과 같다.

```
firstname.lastname@company.com
```

또한 높은 권한 계정도 종종 다음과 같은 **단순한 이름으로 생성되는 경우가 있다.**

```
admin
administrator
root
```

보안 테스트 시에는 다음과 같은 부분을 확인해야 한다.

- 로그인 없이 **user profile 접근 가능 여부**
- HTTP response에서 **email 주소 노출 여부**
- 관리자 또는 IT 계정 정보 노출 여부

이러한 정보는 공격자가 **유효한 username 목록을 구축하는 데 활용**될 수 있다.

---

## Brute-forcing passwords

Password 역시 brute-force 공격의 대상이 된다. 공격의 난이도는 **password strength**에 따라 달라진다.

많은 웹사이트는 다음과 같은 **password policy**를 적용한다.

- 최소 길이 요구
- 대문자 / 소문자 혼합
- 특수문자 포함

하지만 실제 사용자 행동은 종종 보안 정책을 약화시킨다.

예를 들어 사용자는 기억하기 쉬운 패스워드를 **정책에 맞게 변형**하는 경우가 많다.

예시:
```
mypassword → Mypassword1!
mypassword → Myp4$$w0rd
```

또한 **정기적인 password 변경 정책**이 있는 경우에도 사용자는 다음과 같이 **예측 가능한 방식으로 변경**하는 경우가 많다.

```
Mypassword1! → Mypassword1?
Mypassword1! → Mypassword2!
```

이러한 인간의 패턴을 활용하면 brute-force 공격은 **단순한 랜덤 공격보다 훨씬 효율적으로 수행될 수 있다.**

---

## Username enumeration

**Username enumeration**은 웹사이트의 동작 변화를 관찰하여 **유효한 username을 식별하는 공격 기법**이다.

주로 다음과 같은 위치에서 발생한다.

- 로그인 페이지
- 회원가입 페이지

예를 들어:

- **username이 존재하지 않는 경우**
- **username은 맞지만 password가 틀린 경우**

이 두 상황에서 **서버 응답이 다르면 공격자는 유효한 username을 알아낼 수 있다.**

이는 brute-force 공격의 효율을 크게 높인다.

---

## Indicators of username enumeration

Username enumeration은 보통 다음과 같은 차이를 통해 발견할 수 있다.

### 1. HTTP 상태 코드의 차이 (Status code differences)

대부분의 로그인 실패 요청은 동일한 HTTP status code를 반환한다.

하지만 특정 요청에서 **다른 status code가 반환된다면** 이는 username이 유효하다는 신호일 수 있다.

보안적으로 안전한 구현은 다음과 같다.

```
모든 로그인 실패 → 동일한 status code 반환
```

---

### 2. 오류 메세지의 차이 (Error message differences)

웹사이트가 다음과 같은 **서로 다른 에러 메시지**를 반환하는 경우가 있다.

예시:

```
Invalid username
Incorrect password
```

이 경우 공격자는 **username 존재 여부를 쉽게 판단할 수 있다.**

보안적으로 권장되는 방식은 다음과 같다.

```
Invalid username or password
```

즉 **항상 동일한 일반적인 메세지 를 반환해야 한다.**

또한 눈에 보이지 않는 **공백이나 철자 차이**도 enumeration을 가능하게 만들 수 있다.

#### BurpSuite 예제

Lab: [**Username enumeration via subtly different responses**]({% post_url 2026-03-11-#16username enumeration-Lab %})

---

### 3. 응답 반응시간의 차이 (Response time differences)

Response time 역시 중요한 단서가 된다.

예를 들어 서버가 다음과 같은 순서로 인증을 처리한다고 가정해보자.

1. username 존재 여부 확인  
2. password 검증  

이 경우:

- **존재하지 않는 username → 바로 실패**
- **존재하는 username → password 검증 추가 수행**

이 과정 때문에 **유효한 username 요청의 응답 시간이 약간 더 길어질 수 있다.**

공격자는 다음과 같은 방식으로 이 차이를 더 크게 만들 수 있다.

- 매우 긴 password 입력
- 반복 요청을 통한 평균 response time 측정

---

## 정리

Password 기반 인증은 가장 널리 사용되는 인증 방식이지만, 그만큼 공격자에게도 익숙한 구조이다.

따라서 로그인 기능이 세부적으로 안전하게 설계되지 않으면, 단순한 추측 공격만으로도 계정 탈취로 이어질 수 있다.

예를 들어, 공격자는 brute-force 공격으로 비밀번호를 반복 대입할 수 있고,
응답 메시지나 처리 시간의 차이를 통해 유효한 사용자명을 먼저 식별할 수도 있다.

또한 사용자명이 일정한 규칙을 따르거나 비밀번호가 약한 경우, 이러한 공격의 성공 가능성은 더욱 높아진다.

이를 방지하기 위해서는 로그인 시도 횟수 제한, 계정 잠금, 일반화된 에러 메시지, 일관된 상태 코드, 균일한 응답 시간과 같은 보호 장치가 필요하다.
이러한 요소들은 각각 독립적인 방어 수단이면서도, 함께 적용될 때 인증 시스템의 보안성을 크게 높인다.

결론적으로, 안전한 인증 시스템은 단순히 비밀번호를 검증하는 것에 그치지 않고, 공격자가 계정 존재 여부나 인증 실패 원인을 추론할 수 없도록 설계되어야 한다. 그렇지 않으면 자동화된 공격을 통해 인증 우회나 계정 탈취가 현실적인 위협이 될 수 있다.

---

출처: [PortSwigger Acedemy](https://portswigger.net/web-security/authentication/password-based)
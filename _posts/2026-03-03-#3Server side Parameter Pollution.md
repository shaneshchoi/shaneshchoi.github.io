---
title: Server-side parameter pollution (SSPP) 취약점
description: 시스템이 내부적으로 사용하는 API 취약점
categories: [Web Security, API Testing]
tags: [웹 보안]
pin: false
math: true
mermaid: true
toc: true
---

# Server-side parameter pollution이란?

사용자의 입력값이 서버 내부 API 요청에 그대로 삽입되면서 내부 요청의 파라미터를 조작할 수 있게 되는 취약점

## 개념 정리
이 취약점은 다음과 같은 구조에서 user input이 내부 API 요청에 제대로 인코딩 없이 삽입될 때 발생한다:
 
Client → Web server → Internal API

사용자는 외부 웹 서버와 통신하지만, 웹 서버는 다시 내부 API에 요청을 보냄

## Vulnerable Architecture Example
사용자가 아래와 같이 검색 요청을 보낸다고 가정해보자
```text
GET /userSearch?name=shane&back=/home
```
{: file='User to Server'}
웹 서버는 내부 API에 대음과 같은 요청을 생성한다:
```text
GET /user/search?name=shane&publicProfile=true
```
{: file='Server to API'}
여기서 문제는 'name' 파라미터가 그대로 내부 요청에 삽입된다는 점이다.

## SSPP의 위험성
SSPP를 통해 공격자는:
1. 기존 파라미터를 override
2. 새로운 파라미터 추가
3. 서버 요청 일부를 잘라내기
4. 비공개 데이터 접근

등이 가능하다.
예시를 통해 알아보자. 

---

## 1. Query Truncation (잘라내기 공격)

'#' 문자는 URL Fragment를 의미한다.

공격 요청:
```
GET /userSearch?name=shane%23foo&back=/home
```
여기서의 `'%23'`은 `#`의 URL 인코딩 값이다.

URL Encoding없이 `#`을 사용할 경우에 브라우저가 fragment로 처리해서 서버로 보내지 않음.

위 요청은 내부적으로 다음과 같이 변할 수 있다:
```
GET /users/search?name=shane#foo&publicProfile=true
```

이때 `'#'` 이후는 서버로 전달 되지 않을 수 있으므로 `&publicProfile=true` 가 제거 될 수 있다.

결과: 원래는 `publicProfile=true` 라서 공개 프로필만 반환했는데,
공격으로 인해 비공개 프로필도 반환될 수 있음. 


관측 포인트:
1. publicProfile 제한이 사라졌는지
2. 비공개 계정이 반환되었는지
3. 응답 길이의 변화가 있었는지

---

## 2. Injecting New Parameters

'&'를 삽입하면 새로운 파라미터를 추가할 수 있다.

공격 요청:
```
GET /userSearch?name=shane%26foo=xyz&back=/home
```
여기서의 `'%23'`은 `&`의 URL 인코딩 값.

내부 요청:
```
GET /users/search?name=shane&foo=xyz&publicProfile=true
```

관측 포인트:
1. 응답이 동일할 경우 - 무시됨
2. 에러가 발생할 경우 - 파라미터 처리됨
3. 응답 (Response) 구조가 바뀌었을 경우 - 영향 있음

---

## 3. Injecting Valid(Hidden) Parameters

만약 hidden parameter가 있을 경우에,

공격 요청:
```
GET /userSearch?name=shane%26email=foo
```

내부 요청:
```
GET /users/search?name=shane&email=foo&publicProfile=true
```

이런 식으로, 없던 `email` 파라미터를 추가해 검색 로직이 바뀔 수 있음.

---

## 4. Overriding Existing Parameters

이미 외부에 노출된 파라미터를 동일한 이름으로 중복 전달하여 기존 값을 덮어쓰는 공격이다. 예를 들어, 기존에 존재하는 `name` 파라미터를 다시 전달하여 값을 override 할 수 있다.

공격 요청:
```
GET /userSearch?name=shane%26name=evil
```

내부 요청:
```
GET /users/search?name=shane&name=evil&publicProfile=true
```
으로 바뀌게 된다.

이때, Back-end의 서버 기술에 따라 다르게 작동하는데,
1. PHP - 마지막 값 사용 `evil`
2. ASP.NET - 값 결합 (shane, evil) → `Invalid username`
3. Node.js / Express - 첫 번째 값 사용 → `shane`

따라서 PHP 환경이라면 `evil`이 검색될 수 있다.
이 경우, `name=administrator`를 통해 관리자 계정 접근을 시도할 수 있음.

--- 

## Root cause

발생 원인으로는, 백엔드 로직의 취약한 코드가 있다. 사용자 입력을 단순 문자열 결합으로 삽입하게 될 경우에 발생한다. 

예시
```python
internal_url = "users/search?name=" + user_input + "&publicProfile=true"
```
{: file='vulnerable_logic.py'}

---

## 테스트 요약 정리
1. `%23` -> Query truncate
2. `%26` -> 파라미터 추가
3. Valid(Hidden) Parameter 삽입
4. 기존 파라미터 override

테스트 후에,
- 상태 코드, 응답 길이, 에러 메시지, 데이터 변화 등을 관찰.

--- 

## 방어 전략

1. 올바른 URL Encoding
사용자 입력은 반드시 URL 인코딩 처리

2. 문자열 Concatenation 사용지양
Query builder 또는 Parameterized 방식 사용

3. 내부 API 재검증
내부 API에서도 인증 및 권한 검증 수행

4. Input Validation
특수 문자 (`#`,`&`,`=`) 필터링

---

## BurpSuite 예제

Lab #1: [**Exploiting server-side parameter pollution in a query string**]({% post_url 2026-03-03-#4SSPP-Lab1 %})

---

출처: [PortSwigger Academy](https://portswigger.net/web-security/api-testing/server-side-parameter-pollution)
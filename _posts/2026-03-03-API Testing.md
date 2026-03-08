---
title: API Testing 방법 및 방어
description: API의 공격 표면과 주요 취약점 정리
categories: [Web Security, API Testing]
tags: [웹보안, API]
pin: false
toc: true
---

# API Testing이란?

API(Application Programming Interface)는 시스템과 시스템이 통신하는 인터페이스다.

요즘 웹 애플리케이션은 대부분 다음 구조로 동작한다.

```
Front-end → API → Database
```

과거처럼 서버가 완성된 HTML을 내려주는 구조가 아니라,  
브라우저가 API에 요청을 보내고 JSON을 받아 처리하는 방식이 일반적이다.

결국 현대 웹 환경에서의 공격은 API를 중심으로 이루어진다고 봐도 무리가 없다.

---

## 왜 API Testing이 중요한가

API 취약점은 CIA Triad의 세 요소를 직접적으로 위협한다.

- Confidentiality
- Integrity
- Availability

또한 마이크로서비스 아키텍처에서는  
프론트엔드에서 사용하지 않는 API가 따로 존재하는 경우가 많다.

이 때문에 다음과 같은 전통적인 웹 취약점 역시  
API 레이어에서 그대로 발생한다.

- SQL Injection
- IDOR
- Access Control Issues
- Authentication Bypass

즉, 웹 취약점과 API 취약점을 분리해서 생각할 수 없다.

---

## API Testing의 기본 흐름

일반적으로 다음 순서로 진행한다.

```
Recon → Analyze → Exploit
```

여기서 Recon은 Reconnaissance의 약자로,  
테스트 전에 공격 표면을 최대한 수집하는 단계다.

---

## API Recon

### 1. API 문서 탐색

다음과 같은 경로는 항상 확인해볼 필요가 있다.

```
/api
/swagger
/swagger/index.html
/openapi.json
```

Swagger나 OpenAPI 문서가 노출되어 있다면  
엔드포인트, 파라미터, 메소드, 요청 구조까지 한 번에 확인할 수 있다.

Machine-readable 문서는 사실상 명세서이자 설계도다.

예시:

```json
{
  "paths": {
    "/users": {
      "get": {}
    }
  }
}
```

문서가 없다면 브라우징과 크롤링을 통해 직접 찾아야 한다.

---

### 2. Base Path 확인

예를 들어 다음 경로를 발견했다면

```
/api/swagger/v1/users/123
```

상위 경로도 반드시 확인해야 한다.

```
/api/swagger/v1
/api/swagger
/api
```

의외로 상위 경로에서 추가 정보가 노출되는 경우가 많다.

---

### 3. JavaScript 파일 분석

프론트엔드 JS 파일 안에는 직접 호출되지 않은 API가 남아있을 수 있다.

```javascript
fetch("/api/admin/users")
```

Burp의 JS Link Finder 같은 도구를 사용하면 자동 추출도 가능하다.

---

## Endpoint 분석

엔드포인트를 찾았다면 다음을 확인한다.

- 어떤 HTTP Method를 지원하는지
- 어떤 파라미터를 받는지
- 인증이 필요한지
- Rate limit이 있는지
- Content-Type에 따라 동작이 달라지는지

예시:

```text
GET /api/tasks
POST /api/tasks
DELETE /api/tasks/1
```

프론트엔드에서는 GET만 사용하더라도  
DELETE가 열려 있을 가능성은 충분하다.

---

## HTTP Method 테스트

다음 메소드는 기본적으로 확인 대상이다.

- GET
- POST
- PUT
- PATCH
- DELETE
- OPTIONS

OPTIONS 요청으로 허용 메소드를 확인할 수 있다.

다만 실제 운영 환경에서는  
중요한 리소스를 대상으로 무분별한 테스트를 하면 안 된다.

---

## Content-Type 변조

API는 특정 Content-Type을 기대하는 경우가 많다.

대표적인 타입은 다음과 같다.

- application/json
- application/xml
- multipart/form-data
- x-www-form-urlencoded

Content-Type을 바꿨을 때 에러 메시지가 달라지거나  
검증 로직이 달라지는 경우가 있다.

예를 들어 JSON 요청은 안전하게 처리하지만  
XML 요청은 별도 검증 없이 처리하는 경우도 존재한다.

이 차이를 이용하면 필터 우회나 Injection이 발생할 수 있다.

---

## Hidden Endpoint 찾기

예를 들어 다음 endpoint를 발견했다고 가정하자.

```
PUT /api/user/update
```

그렇다면 update 부분을 다른 동사로 바꿔보는 식의 테스트가 가능하다.

```
/api/user/delete
/api/user/add
/api/user/reset
/api/user/promote
/api/user/admin
```

응답 코드에 따라 존재 여부를 유추할 수 있다.

- `404`: 존재하지 않음
- `401`/`403`: 존재하지만 인증 필요
- `405`: 메소드는 잘못됐지만 endpoint는 존재
- `200`: 정상 응답

이 과정에서 naming convention을 이해하는 것이 중요하다.

---

## Hidden Parameter 찾기

요청이 다음과 같을 때

```json
{
  "username": "shane",
  "email": "shane@example.com"
}
```

서버 내부 객체는 더 많은 필드를 가지고 있을 수 있다.

예:

```
isAdmin=true
role=admin
debug=true
accessLevel=10
```

파라미터를 추가했을 때 응답이 달라지는지 확인한다.  
에러 메시지, 상태 코드, 반환 데이터의 변화가 힌트가 된다.

---

## Mass Assignment

Mass Assignment는 자동 객체 바인딩으로 인해 발생하는 취약점이다.

PATCH 요청:

```json
{
  "username": "shane",
  "email": "shane@example.com"
}
```

GET 응답:

```json
{
  "id": 159753,
  "name": "shane",
  "email": "shane@example.com",
  "isAdmin": false
}
```

내부 객체에 `isAdmin` 필드가 존재한다면  
검증이 없다면 다음과 같은 요청이 가능해진다.

```json
{
  "username": "shane",
  "email": "shane@example.com",
  "isAdmin": true
}
```

이 경우 권한 상승이 발생한다.

탐지 방법으로는,

1. GET 요청으로 객체 구조 확인  
2. PATCH 요청과 비교  
3. 존재하는 필드를 임의 값으로 전송 후 반응 관찰  

Spring, Rails, Express, Django 등  
여러 프레임워크에서 자동 바인딩이 기본 동작이다.

---

## 방어 전략

### 1. API 문서 보호

- Swagger를 외부에 노출하지 않는다
- 내부 문서는 인증 뒤에 둔다
- 운영 환경에서 불필요한 OpenAPI 파일 제거  

### 2. HTTP Method Allowlist

허용된 메소드만 처리하고  
나머지는 405를 반환하도록 구성한다.  

### 3. Content-Type 검증

엔드포인트별 허용 타입을 고정하고  
예상하지 않은 타입은 415로 거부한다.  

### 4. Generic Error Message

구체적인 SQL 에러나 내부 경로를 노출하지 않는다.  
상세 오류는 내부 로그에만 기록한다.  

### 5. 모든 API 버전 보호

```
/api/v1/users
/api/v2/users
```

구버전 API를 방치하면 우회 경로가 된다.  
사용하지 않는 버전은 제거하는 것이 가장 안전하다.  

### 6. Mass Assignment 방지

- 허용 필드만 명시적으로 바인딩
- 내부 Entity 직접 사용 금지
- Update 전용 DTO 사용
- isAdmin, role 등 민감 필드는 서버 로직에서만 수정

---

## BurpSuite 예제

Lab #1: [**Exploiting an API endpoint using documentation**]({% post_url 2026-03-04-API Testing-Lab1 %})  
Lab #2: [**Finding and exploiting an unused API endpoint**]({% post_url 2026-03-05-API Testing-Lab2 %})
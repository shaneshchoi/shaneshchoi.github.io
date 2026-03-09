---
title: JWT 공격이란 무엇일까?
description: JWT가 어떻게 사용되고 악용될 수 있는지 알아보자.
categories: [Web Security, JWT]
tags: [웹 보안, JWT]
toc: true
---

# JWT 공격

웹 애플리케이션에서 인증이나 세션 관리를 구현할 때 **JWT(JSON Web Token)** 를 사용하는 경우가 많다.  
특히 API기반 서비스나 여러 서버가 분산된 환경에서 많이 사용되는데, 서버가 별도의 세션을 저장하지 않아도 된다는 장점이 있기 때문.

하지만 이 구조 때문에 JWT 처리 방식이 조금만 잘못 구현되어도 심각한 보안 문제가 발생할 수 있다.  
대표적으로 공격자가 **토큰의 내용을 조작**해 다른 사용자를 가장하거나 권한 상승 (Privilege escalation) 공격이 가능하다. 

---

## JWT가 뭘까?

JWT는 **JSON 형태의 데이터를 안전하게 전달하기 위한 토큰 포맷** 이다.

주로, 
- 인증 (Authentiation)
- 세션 관리 (Session Management)
- 접근 제어 (Access Control)와 같은 용도로 사용된다.

JWT의 가장 큰 특징은 서버가 세션 상태를 따로 저장하지 않는다는 점이다 (필요한 사용자 정보가 토큰 자체에 들어 있기 때문)

즉, 서버는 요청을 받을 때마다 DB에서 세션을 조회하는 대신, 클라이언트가 보내온 JWT를 검증한 뒤 그 안의 데이터를 그대로 사용한다.

이 구조 덕분에 여러 서버가 분산된 환경에서도 쉽게 인증 처리가 가능하지만, 동시에 토큰 자체가 공격의 대상이 될 수 있다는 문제도 생긴다.

---

## JWT 구조에 대해 알아보자

JWT는 다음과 같이 **세 개**의 부분으로 구성되며, 각 부분은 `.`으로 구분된다.

```
header.payload.signature
```

예시: 
```
eyJraWQiOiI5MTM2ZGRiMy1jYjBhLTRhMTktYTA3ZS1lYWRmNWE0NGM4YjUiLCJhbGciOiJSUzI1NiJ9
.
eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTY0ODAzNzE2NCwibmFtZSI6IkNhcmxvcyBNb250b3lhIiwic3ViIjoiY2FybG9zIiwicm9sZSI6ImJsb2dfYXV0aG9yIn0
.
SYZBPIBg2CRjXAJ8vCER0LA_ENjII1JakvNQoP-Hw6GG1zfl4JyngsZReIfqRvIAEi...
```

### Header - 토큰의 메타데이터
```JSON
{
    "alg":"RS256",
    "typ":"JWT"
}
```
여기서 **alg**는 토큰이 어떤 알고리즘으로 서명되었는지를 나타낸다.

### Payload - 전달 데이터 (claims)를 포함
```JSON
{
  "iss": "portswigger",
  "exp": 1648037164,
  "name": "Shane Choi",
  "sub": "Shane",
  "role": "blog_author"
}
```

여기에는 보통
- 사용자 ID
- 이메일
- 권한 (role)
- 토큰 만료시간 (exp) 등을 포함한다.

중요한 점은 이 데이터는 **암호화된 것이 아니라** 단순히 인코딩 되어 있다는 것.  
따라서 누구나 쉽게 디코딩해서 내용을 확인할 수 있다.

### Signature - 서명

마지막 부분은 **서명** 이다.

```
sign(
    base64(header) + "." + base64(payload),
    secret_key
)
```

코드에서 확인할 수 있듯, *header*와 *payload*를 합친 뒤, 서버의 `secret_key`로 서명 한다.
따라서 토큰의 무결성 (Integrity)를 검증할 수 있는것이다.

만약 *payload*의 값을 한 글자라도 수정하면  
서명 값이 달리지기 때문에 서버에서 검증이 실패한다.

---

## JWT, JWS, JWE 차이가 뭘까?

JWT는 사실 데이터 포맷 자체만 정의한 개념이다.

실제로는 다음 **두 가지** 방식으로 구현된다.

- JWS :데이터를 서명 (Signature)
- JWE :데이터를 암호화 (Encryption)

대부분의 웹 애플리케이션에서 사용하는 JWT는 **JWS**형태다.
즉, 암호화가 아니라 서명만 되어 있는 토큰 이다.

그래서 *payload*내용은 누구나 볼 수 있지만,  
서명을 통해 데이터 변조 여부만 확인하는 구조다.

---

## JWT 공격이 가능한 이유

JWT 기반 인증 시스템의 핵심 전제는 **"토큰의 서명만 올바르면 *payload*데이터를 신뢰 가능하다"**는 점이다.

하지만 현실에서는 다음과 같은 문제들이 발생한다:
1. 서버가 서명을 제대로 검증하지 않는 경우
2. JWT 라이브러리를 잘못 사용하는 경우
3. secret key가 유출되거나 추측 가능한 경우
4. 알고리즘 처리 방식에 구현 오류가 있는 경우

이런 상황에서는 공격자가 토큰을 직접 조작하더라도, 서버가 이를 정상 토큰으로 받아들일수 있다.

예를 들어 다음과 같은 JWT가 있다고 가정해보자.
```json
{
  "username": "Shane",
  "isAdmin": false
}
```

정상적인 구현이라면 서명 검증 단계에서 차단되어야 하지만,

만약 서버가 *payload*값을 그대로 admin 권한 판단에 사용한다면,  
`isAdmin`값을 `true`로 바꾸는 것만으로도 권한 상승이 가능하게 된다.

이후 글에서 보겠지만  
**JWT 검증 로직**이 잘못 구현된 경우 생각보다 쉽게 우회가 가능하다.

---

예제를 통해 서명 검증을 하지 않는 JWT의 취약점을 간단히 알아보자:

### 예제 1

![설명](/assets/burp_images/JWT/jwt1_1.png){:width="1200px"}
> 순서대로 *header*, *payload*, 그리고 나머지 *signature*

<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt1_2.png" alt="설명" width="400">
</div>

*payload* 부분을 Base64로 decode해보면, `"sub":"shane"` 으로 디코딩된걸 확인 할 수 있다.
만약 서버가 서명 검증 단계에서 차단하지 않고 *payload*값을 그대로 사용한다면, `"sub":"administrator"` 혹은 `"sub":"admin"` 등으로 바꾸는 것만으로도 권한 상승이 가능해진다. 

---

### 예제 2

<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt1_3.png" alt="설명" width="400">
</div>
<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt1_4.png" alt="설명" width="400">
</div>

`"sub":"administrator"`로 변경했지만 여전히 `401 unauthorized` 응답을 반환.

이번에는 JWT의 *header*를 조작 해보자.

<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt1_5.png" alt="설명" width="400">
</div>

`"alg": "RS256"`을 사용중인걸 볼 수 있는데, 해당 값을 `"none"`으로 수정.

```json
{
    "alg": "none"
}
```
> 여기서 `alg: none`은 **"이 토큰은 서명이 없는 JWT"** 라는 의미.

정상적으로 구현된 서버는 이런 토큰을 거부해야 하나, 취약할 경우 서명 검증을 건너 뛰게끔 설정되어있을 수 있다.  
예시:
```
if (alg == none)
    skip signature verification
```

`alg: none`으로 설정할 경우, *signature*가 존재하면 안되기 때문에 JWT의 구성인 *header.payload.signature*에서 *signature*를 제외한  
*header.payload.*만 전송하도록 한다. **JWT 형식을 유지하기 위해서 마지막 `.`은 남겨야함** 

따라서, 다음과 같은 요청을 전송하게 된다
![설명](/assets/burp_images/JWT/jwt1_6.png){:width="1200px"}


---

JWT 공격은 단순히 토큰을 수정하는 것에서 끝나지 않는다.
구현 방식에 따라 다양한 취약점이 발생할 수 있는데, 대표적으로는 

- 서명 검증하지않는 JWT
- `alg:none` 취약점
- 알고리즘 혼돈 공격 (algorithm confusion)
- weak secret key brute-force 공격
- `kid` 헤더 기반 공격
등이 있다.

다음 글에서는 실제로 JWT 검증이 잘못 구현된 경우 어떻게 인증을 우회할 수 있는지를 하나씩 살펴보려고 한다.

---

출처: [PortSwigger Academy](https://portswigger.net/web-security/jwt)
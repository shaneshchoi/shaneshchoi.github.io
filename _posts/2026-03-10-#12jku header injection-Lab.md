---
title: JWT attacks 실습(3) - JWT authentication bypass via jku header injection (PortSwigger Academy)
description: JWT header의 jku 파라미터를 이용해 공격자의 JWK Set을 사용하도록 유도하여 관리자 권한을 획득하는 공격
categories: [Web Security, JWT, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: JWT authentication bypass via jku header injection

Lab Link: [JWT authentication bypass via jku header injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jku-header-injection)

---

## 핵심 포인트

이 lab의 취약점은 다음과 같다.

- 서버가 JWT header의 **`jku` 파라미터를 지원**
- `jku` 값으로 지정된 URL에서 **JWK Set을 다운로드**
- 해당 URL이 **신뢰된 도메인인지 검증하지 않음**

즉 공격자는 **자신이 만든 JWK Set을 서버가 사용하도록 유도**할 수 있다.

이번 실습에서는 **exploit server에 JWK Set을 업로드**하고  
JWT header의 `jku` 값을 해당 URL로 변경하여 공격을 수행한다.

---

## Solution

### 1. JWT 확인

먼저 `wiener:peter` 계정으로 로그인한다.

로그인 이후 요청을 확인하면 session cookie에 JWT가 포함되어 있다.

Proxy → HTTP History → 로그인 이후 요청 확인

![설명](/assets/burp_images/JWT/jwt_lab3_0-0.png){:width="1200px"}

![설명](/assets/burp_images/JWT/jwt_lab3_0-1.png){:width="1200px"}

로그인 이후 요청에는 다음과 같은 **JWT session cookie**가 포함되어 있다.

---

### 2. RSA key 생성

Burp의 JWT Editor를 이용해 공격에 사용할 **RSA key pair**를 생성한다.

JWT Editor → Keys → New RSA Key

<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt_lab3_1-0.png" alt="설명" width="500">
</div>

Generate 버튼을 눌러 **RSA key pair**를 생성한다.

이 key는 이후 **JWT 서명에 사용되는 private key**이며  
exploit server에는 **public key만 업로드**하게 된다.

---

### 3. Exploit server에 JWK Set 생성

브라우저에서 랩 내에 있는 **Exploit Server**를 연다.

File명을 `/jwks.json`으로 수정하고,  
헤드 영역에 Content-Type을 `application/json`으로 수정한다.

<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt_lab3_1-1-0.png" alt="설명" width="500">
</div>

`Body`역역에 keys set을 만들고,  
방금전 JWT Editor로 생성한 RSA key pair의 **공개키(Public key)**를 복붙한다.

<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt_lab3_1-1.png" alt="설명" width="500">
</div>

이후 **Store**버튼을 눌러 exploit을 저장하게되면,  
exploit server는 공격자가 만든 JWK Set을 제공하는 서벼 역할을 하게 된다.

### 4. JWT 수정

Burp Repeater에서 `/admin`페이지를 접근하기 전에, 몇 가지 수정사항이 필요하다.

![설명](/assets/burp_images/JWT/jwt_lab3_1-2.png){:width="1200px"}

우선 기존 `kid`값을 **exploit server**에 업로드한 JWK의 kid 값으로 변경, JWT header에 `jku` 파라미터를 추가해준다.

`jku`값에는 우리가 **exploit server**에 업로드한 JWK Set의 URL을 입력 한다.

이렇게 하면 서버는 JWT 검증 시 해당 URL에서 JWK Set을 가져오게 된다.

<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt_lab3_1-3.png" alt="설명" width="500">
</div>

이후에 JWT payload에서 `sub`값을 `administrator`로 수정 하고 하단 Sign 버튼으로 JWT를 재서명하게 되면,  
JWT는 공격자가 만든 **개인키(Private key)**로 서명 된다.

### 5. carlos 계정 삭제

`/admin`에 해당 request를 요청하면 이전 실습과 동일하게 carlos의 계정을 삭제하는 URL을 반환한다:   
`/admin/delete?username=carlos` 

해당 요청을 보내면서 lab은 마무리 된다.

---

## 정리

이 취약점의 핵심은 **JWT header**의 `jku` 값을 그대로 신뢰한다는 점이다.

정상적인 구현이라면 서버는 신뢰된 인증 서버의 **JWK Set**만 허용해야 한다.  
하지만 이 lab에서는 임의의 URL에서 key를 가져오는 것이 가능하다.

결과적으로 공격자는 다음 과정을 통해 인증을 우회할 수 있었다.

1. 공격자가 RSA key pair 생성
2. public key를 포함한 JWK Set을 exploit server에 업로드
3. JWT header의 `jku` 값을 공격자 서버로 변경
4. attacker private key로 JWT 서명

이렇게 하면 서버는 공격자의 public key를 이용해 서명을 검증하게 되고 공격자가 만든 JWT를 정상 토큰으로 인식하게 된다.
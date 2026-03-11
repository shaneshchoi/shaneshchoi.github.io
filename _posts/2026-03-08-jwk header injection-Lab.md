---
title: JWT attacks 실습(2) - JWT authentication bypass via weak jwk header injection
description: JWT header의 jwk 파라미터를 이용해 공격자의 공개키로 서명 검정을 우회하는 공격
categories: [Web Security, JWT, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: JWT authentication bypass via weak jwk header injection

Lab Link: [JWT authentication bypass via weak jwk header injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection)

---

## 핵심 포인트

이 lab의 취약점은 다음과 같다.

- 서버가 JWT header의 **`jwk` 파라미터를 지원**
- JWT 내부에 포함된 **공개키를 그대로 신뢰**
- 검증에 사용할 **공개키의 출처를 검증하지 않음**

공격자는 자신의 **RSA key pair**를 생성한 후,  
자신의 **개인키**로 JWT를 서명, **공개키**를 `jwk` 헤더에 삽입하여  
서버가 이를 검증하도록 유도 할 수 있다.

JWT Tools나 Python으로도 공격이 진행하지만,  
이번 실습에서는 BurpSuite의 **JWT Editor extension**을 이용해 공격 하는 방법을 알아보자.

- 원하는 payload 작성
- 정상적인 signature 생성
- 관리자 권한 위조

---

## Solution

### 1. JWT 확보

먼저 `wiener:peter` 계정으로 로그인해서 session cookie를 확인한다.

Proxy → HTTP History → Request

![설명](/assets/burp_images/JWT/jwt_lab2_0-0.png){:width="1200px"}

![설명](/assets/burp_images/JWT/jwt_lab2_0-1.png){:width="1200px"}

로그인 이후 요청에는 다음과 같은 **JWT session cookie**가 포함되어 있다.

```
Cookie: session=eyJraWQiOiJkZmFjNzkyMy1mNTFlLTQxOWMtOGM4Ni1mNWVjZjlmY2U1NmYiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MzEwNzY3Niwic3ViIjoid2llbmVyIn0.hBUkHULBZP8llTYz-9biCSkLOkjxiZSA__HJc5FcFNq1h-HxmlpmSaNDlnhAevR6kcQRqiQKAGCDsl4vgx2JJOw_jvEJ0LdnPukeFVXD5148ihctW_yWikyK4gJiYh62kEWDPrs2HzPOutiV7s8yCQ-DORawxjxpsmmGY38b9Sublcfckgr1ReVEJXsXH5lFuYsoJIzUesFHRr1JF4BDcY7BnYFg5ldE6JhdpnaS2VQk2FM37urONvJghownfVzGqABfCQRBQhwe_vhBZkzGIBycpKArTl4LMevooBAsfWexmxvwJKuLzi8n2lScB5uTeJ9wjZxnNNnJPt8y_iULug
```

---

### 2. RSA key 생성

Burp의 JWT Editor를 이용해 공격에 사용할 **RSA key pair**를 생성한다.

JWT Editor → Keys → New RSA Key

이후 **Generate**버튼을 눌러 RSA key pair를 생성한 뒤 저장한다.

![설명](/assets/burp_images/JWT/jwt_lab2_1-0.png){:width="1200px"}

---

### 3. JWT Payload 수정 및 jwk injection

Burp Repeater에서 JWT Editor 탭을 열고,  
*payload*의 `sub`값을 `administrator`로 수정.

![설명](/assets/burp_images/JWT/jwt_lab2_1-1.png){:width="1200px"}

하단에서 Attack → Embedded JWK 선택 후 방금 *JWT Editor*를 통해 만든 RSA Key와 RS256 알고리즘 선택한다.

이렇게 되면 
- 공격자의 **개인키**로 JWT를 재서명
- 공격자의 **공개키**를 `jwk` 헤더에 삽입  
과정이 이루어 진다.

JWT header 섹션을 보면 `jwk`값이 추가된 것을 확인할 수 있다.

![설명](/assets/burp_images/JWT/jwt_lab2_1-2.png){:width="1200px"}

---

### 4. GET /admin 요청

수정된 JWT로 `/admin` 페이지에 요청을 전송하면 Respose 200을 반환,  
Admin Panel에 접속이 가능한 것을 볼수 있다.

![설명](/assets/burp_images/JWT/jwt_lab2_2-0.png){:width="1200px"}
> 요청

![설명](/assets/burp_images/JWT/jwt_lab2_2-1.png){:width="1200px"}
> 응답

---

### 5. carlos 계정 삭제

`/admin` 응답에서 carlos의 계정을 삭제하는 URL을 반환,

![설명](/assets/burp_images/JWT/jwt_lab2_2-2.png){:width="1200px"}

해당 URL로 요청을 전송하면서 lab은 마무리 된다.

---

## 정리

이 취약점의 핵심은 **JWT header**의 `jwk` 값을 그대로 신뢰한다는 점이다.

정상적인 구현에서는 서버가 미리 신뢰된 **개인키(public key)**만 사용해 서명을 검증해야 한다.

하지만 이 lab에서는 JWT에 포함된 `jwk` 값을 그대로 검증 키로 사용하기 때문에 공격자가 자신의 key pair로 JWT를 생성해 인증을 우회할 수 있었다.
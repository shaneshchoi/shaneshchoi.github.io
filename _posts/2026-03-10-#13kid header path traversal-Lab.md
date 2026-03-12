---
title: JWT attacks 실습(4) - JWT authentication bypass via kid header path traversal
description: JWT header의 kid 파라미터를 이용한 path traversal 공격으로 verification key를 조작하여 관리자 권한을 획득하는 공격
categories: [Web Security, JWT, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: JWT authentication bypass via kid header path traversal

Lab Link: [JWT authentication bypass via kid header path traversal](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-kid-header-path-traversal)

---

## 핵심 포인트

이 lab의 취약점은 다음과 같다.

- 서버가 JWT header의 **`kid` 값을 이용해 verification key를 조회**
- 해당 값을 **파일 시스템 경로로 사용**
- `kid` 값에 대한 **검증이 없음**

즉 공격자는 `kid` 값에 **directory traversal**을 삽입해 서버가 사용할 verification key를 조작할 수 있다.

이번 실습에서는 Linux 시스템에 존재하는 **`/dev/null`** 파일을 이용한다.

`/dev/null`을 읽으면 항상 **빈 문자열(empty string)** 이 반환되기 때문에  
이를 이용해 **빈 문자열을 secret key로 사용하도록 서버를 유도**할 수 있다.

---

## Solution

### 1. JWT 확인

먼저 `wiener:peter` 계정으로 로그인한다.

Proxy → HTTP History에서 로그인 이후 요청을 확인하면  
**JWT session cookie**가 포함되어 있다.

(이전 실습 포스팅에서 초기 burp 조작에 대해 언급했으니 생략 하도록 하겠다.)

---

### 2. Signing key 생성

Burp의 **JWT Editor**를 이용해 공격에 사용할 key를 생성한다.

JWT Editor → Keys → **New Symmetric Key**

<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt_lab4_1.png" alt="설명" width="500">
</div>

Generate 버튼을 눌러 key를 생성한 뒤  
`k` 값을 empty string으로 서명을 해줘야한다. 하지만, JWT Editor extension이 empty string으로 서명하는 것을 허용하지 않기 때문에 `null byte`로 우회하여 기입해줘야 한다.

또한, JKW에서 symmetric key (`k`)는  **raw byte**를 **Base64로 인코딩**한 값으로 저장되기 때문에 `null byte`또한 **Base64**로 변환하여 작성해주자.

CyberChef 링크: [CyberChef](https://gchq.github.io/CyberChef/)

<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt_lab4_0.png" alt="설명" width="500">
</div>

`\x00`은 `null byte`를 뜻하는 Hex character이다. 해당 값을 Base64로 인코딩 해주면 `AA==`라는 output이 생긴다.

---

### 3. JWT 수정

Burp Repeater에서 `/admin` 요청을 열고 JWT를 수정한다.

![설명](/assets/burp_images/JWT/jwt_lab4_2.png){:width="1200px"}

먼저 JWT header의 `kid` 값을 다음과 같이 변경한다.  
`../../../../dev/null`

이 값은 **path traversal**을 이용해 서버가 루트 디렉터리에서부터 `/dev/null`파일을 읽도록 만든다.

이후 *payload에서 `sub`값을 `administrator`로 변경한다.

마지막으로 Sign버튼을 눌러 JWT를 재서명 한다.

---

### 4. carlos 계정 삭제

이전 실습과 마찬가지로, `/admin/delete?username=carlos`로 요청을 보내면서 lab은 마무리 된다.

---

## 정리

이 취약점의 핵심은 JWT header의 kid 값을 그대로 신뢰한다는 점이다.

서버가 kid 값을 이용해 verification key 파일을 직접 읽도록 구현되어 있을 경우
공격자는 directory traversal을 이용해 서버가 읽을 key 파일을 조작할 수 있다.

이번 lab에서는 /dev/null 파일을 이용해 빈 문자열을 secret key로 사용하도록 서버를 유도했고 이를 이용해 임의의 JWT를 생성하여 인증을 우회할 수 있었다.
---
title: JWT attacks 실습(1) - JWT authentication bypass via weak signing key (PortSwigger Academy)
description: Weak JWT signing key를 brute-force하여 관리자 권한을 획득하는 공격
categories: [Web Security, JWT, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: JWT authentication bypass via weak signing key

Lab Link: [JWT authentication bypass via weak signing key](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-key)

---

## 핵심 포인트

이 lab의 취약점은 다음과 같다.

- JWT가 **HS256 (symmetric key)** 알고리즘 사용
- secret key가 **매우 약함**
- JWT 하나만 확보하면 **offline brute-force 가능**

secret key를 알아내면 공격자는 다음이 가능하다.

- 원하는 payload 작성
- 정상적인 signature 생성
- 관리자 권한 위조

이번 실습은 BurpSuite를 사용하는 방법과 사용하지 않는 방법 두 가지 모두 알아볼 예정이다.  

실습에 앞서, 준비가 필요하다

### 1. Wordlist 준비

이번 실습에서는 아래 wordlist를 사용한다:
[https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list)

### 2. JWT Editor Extension 설치 (BurpSuite 사용할 경우)

Burp BApp - JWT Editor를 설치한다.

---

## Solution

### 1. JWT 확보

먼저 `wiener:peter` 계정으로 로그인해서 session cookie를 확인한다.

**Option1: 개발자 도구로 확인**

개발자 도구 → Cookie → Session value
![설명](/assets/burp_images/JWT/jwt_lab1_Manual0-0.png){:width="1200px"}

**Option2: BurpSuite 요청 확인**

Proxy → HTTP History → Request

![설명](/assets/burp_images/JWT/jwt_lab1_Burp0-0.png){:width="1200px"}

![설명](/assets/burp_images/JWT/jwt_lab1_Burp0-1.png){:width="1200px"}

각 요청에는 다음과 같은 **JWT session cookie**가 포함되어 있다.

```
Cookie: session=eyJraWQiOiI1ODE3YjIzYi1jNTVmLTRlMDUtOThjNC00MDMxOWMzNmFkM2QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MzAwMDc1Mywic3ViIjoid2llbmVyIn0.Ht3NwrnsdxLD7Q2Pua6klgpsJAS3f_cz0SzaLsfUxH8
```

이 JWT를 복사한다.

---

### 2. Secret key brute-force

JWT secret key는 **hashcat**을 이용해 brute-force 할 수 있다.

**Hashcat**은 GPU 기반 password cracking/hash craking 도구다.  
여기서는 JWP signature를 생성한 secret key를 찾기 위해 사용한다. 

포맷:
`hashcat [공격 방식] [해시 타입] [타겟] [wordlist]`

예시:
`hashcat -a 0 -m 16500 <JWT> jwt.secrets.list`

`-a` 옵션은 **attack mode**,
`0`은 **directory attack (wordlist attack)**,      
`-m` 옵션은 **hash type**,  
`16500`은**JWT 전용 모드**다.

hashcat은 wordlist의 값을 secret key로 사용해 JWT를 다시 서명한 뒤  
기존 signature와 비교한다.

따라서, 우리가 복사했던 JWT와 wordlist를 이용해 다음 명령어를 입력해준다:
`hashcat -a 0 -m 16500 eyJraWQiOiI1ODE3YjIzYi1jNTVmLTRlMDUtOThjNC00MDMxOWMzNmFkM2QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MzAwMDc1Mywic3ViIjoid2llbmVyIn0.Ht3NwrnsdxLD7Q2Pua6klgpsJAS3f_cz0SzaLsfUxH8 ~/Desktop/portswigger/jwt_bruteforce/jwt.secrets.list`

<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt_lab1_1.png" alt="설명" width="400">
</div>

결과:
`eyJraWQiOiI1ODE3YjIzYi1jNTVmLTRlMDUtOThjNC00MDMxOWMzNmFkM2QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MzAwMDc1Mywic3ViIjoid2llbmVyIn0.Ht3NwrnsdxLD7Q2Pua6klgpsJAS3f_cz0SzaLsfUxH8:secret1`  

`secret1`으로 서명한것을 발견할 수 있다.

---

### 3. signing key 생성 (BurpSuite 사용시)

Burp BApp 에서 설치해둔 JWT Editor를 사용해 signing key를 생성해보자.

우선 Burp Decoder 에서 발견된 `secret1` 키를 Base64 기반 encoding 진행해준다.
*secret1 → c2VjcmV0MQ==*

Burp에서 JWT Editor extension을 사용해 새로운 key를 생성한다.
*JWT Editor → Keys → New Symmetric Key*


<div class="img-left-block">
  <img src="/assets/burp_images/JWT/jwt_lab1_Burp2-1.png" alt="설명" width="600">
</div>

위와 같이 생성된 key에서 `k`의 value를 `c2VjcmV0MQ==`로 수정한다.

---

### 4-1. JWT payload 수정 (Manual)

**jwt.io 에서 JWT Encoder 사용하기**

Link: [https://www.jwt.io/](https://www.jwt.io/)  

*jwt.io → JWT Decoder*

![설명](/assets/burp_images/JWT/jwt_lab1_2-1.png){:width="1200px"}

우리가 복사했던 JWT를 **JSON WEB TOKEN**에 복사해주고  
발견했던 `secret1` 키를 **SECRET KEY**에 입력해준다.

이후에 *JWT Encoder* 탭에서 

```json
{
  "sub": "wiener"
}
``` 
데이터 값을
```json
{
  "sub": "administrator"
}
```
로 바꿔준다.

![설명](/assets/burp_images/JWT/jwt_lab1_2-2.png){:width="1200px"}

생성된 *JSON WEB TOKEN* 값을 복사해서 `/admin` 에 접속,  
**개발자 도구**에 있는 session cookie 값을 해당 *JSOB WEB TOKEN*으로 바꿔주면 공격에 성공한다.

### 4-2. JWT Payload 수정 (BurpSuite 사용시)

Burp Repeater에서 `/admin` 요청을 열고 `wiener`를 `administrator`로 JWT payload를 수정,  
JSON Web Token 탭에서 *Sign → 생성한 key 선택*. 

![설명](/assets/burp_images/JWT/jwt_lab1_Burp2-2.png){:width="1200px"}

이렇게 하면 수정된 payload에 대한 정상적인 signature가 생성된다.  
요청을 전송하면 관리자 패널에 접근할 수 있다.

---

### 5. Carlos 계정 삭제

URL에서 수동으로, 혹은 BurpSuite 응답에서 `/admin/delete?username=carlos`를 전송해 계정을 삭제 한다.

---

## 정리

이 취약점의 핵심은 **weak JWT signing key**이다.

JWT는 *payload*가 클라이언트에 저장되기 때문에 signature 검증이 보안의 핵심 요소가 된다.  
하지만 secret key가 약한 경우에 공격자는 이를 brute-force 하여 임의의 JWT를 생성하고 인증을 우회할 수 있다.
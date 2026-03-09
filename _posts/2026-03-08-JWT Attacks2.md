---
title: JWT 취약점과 공격 방식
description: JWT 기반 인증에서 발생할 수 있는 주요 취약점과 공격 방식들을 정리한다.
categories: [Web Security, JWT]
tags: [웹 보안, JWT, PortSwigger]
toc: true
---

# JWT 취약점과 공격 방식

이전 글에서는 JWT가 어떤 구조로 이루어져 있으며, 왜 공격이 가능한 구조인지 살펴보았다.

JWT 기반 인증 시스템은 서버가 세션 상태를 따로 저장하지 않고 **토큰 자체의 서명(Signature)을 검증한 뒤 *payload* 데이터를 신뢰하는 방식**으로 동작한다.  
따라서 이 서명 검증 과정이 잘못 구현되거나, 사용되는 키가 약한 경우 공격자가 토큰을 조작하여 인증을 우회할 수 있다.

이번 글에서는 실제로 자주 발견되는 **JWT 공격 방식들**을 몇 가지 살펴보려고 한다.

---

## Weak Secret Key 공격 (Brute-Force)

JWT는 서명 알고리즘에 따라 **대칭키 (Symmetric Key)**를 사용하는 경우가 있다.

대표적으로,
- HS256
- HS384
- HS512
와 같은 알고리즘이 있는데, 이 방식들은 `HMAC(secret_key, header + payload)`같은 형태로 서명이 생성된다.

즉, `secret_key` 하나로 서명을 생성하고 검증한다.

문제는 이 secret key가 비밀번호처럼 취급되어야 한다는 점이다.

하지만 실제 환경에서는 다음과 같은 실수가 자주 발생한다.
- 기본 예제 코드의 secret key를 그대로 사용
- `"secret"`같은 짧은 문자열 사용
- 개발 과정에서 사용한 placeholder key를 그대로 배포

이 경우 공격자는 **JWT 서명을 brute-force**하여 secret key를 찾아낼 수 있다.  
만약 공격자가 secret key를 알아낸다면 공격자는 원하는 *payload*를 만들어 정상적인 서명을 생성할 수 있게 된다.

```
{
  "sub":"administrator"
}
```

같은 토큰을 직접 만들어도 서버는 정상적인 JWT로 인식하게 되는것.


### Hashcat을 이용한 secret key brute-force

JWT secret key brute-force에는 보통 **hashcat**을 사용한다.

공격에 앞서 필요한 것 두가지:
- 서버에서 발급된 JWT
- secret wordlist

예시로, `hashcat -a 0 -m 16500 <jwt> <wordlist>` 명령어를 살펴보면,

hashcat은
1. wordlist의 문자열을 secret key로 가정
2. JWT header + payload를 재서명
3. 기존 signature와 비교

만약 일치하는 값이 발견되면 `<jwt>:<secret_key>`와 같이 출력된다.

#### BurpSuite 예제

Lab: [**JWT authentication bypass via weak signing key**]({% post_url 2026-03-08-Hashcat-Lab %})

## JWT Header Parameter Injection

JWT header에는 `alg` 외에도 여러 파라미터가 존재할 수 있다.  
ex: `jwk`, `jku`, `kid`.

이 값들은 **서버가 어떤 키를 사용해서 JWT 서명을 검증해야 하는지 알려주는 역할**을 한다.

문제는 이 값들이 토큰 내부에 존재하는 사용자 입력 데이터 (User input data)라는 점.  
즉 공격자가 이 값을 조작할 수 있다.

### jwk 파라미터를 통한 injection

**JWK (JSON Web Key)**는 암호화 키를 JSON 형태로 표현하는 표준 형식이다.
JWT에서는 `jwk` 헤더를 통해 서명을 검증할 때 사용할 **공개키(Public key)**를 토큰 내부에 포함시킬 수 있다.

예시:
```json
{
  "kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG",
  "typ": "JWT",
  "alg": "RS256",
  "jwk": {
    "kty": "RSA",
    "e": "AQAB",
    "kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG",
    "n": "yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9m"
  }
}
```
여기서 `jwk` object는 **RSA 공개키를 JSON 형식으로 표현한 것**이다.

JWT가 `RS256`과 같은 **비대칭키(Asymmetric key)**기반 알고리즘을 사용할 경우,  
다음과 같은 방식으로 서명과 검증이 이루어진다.

- **개인키(Private Key) → JWT 서명**
- **공개키(Public Key) → JWT 서명 검증**

일반적인(정상적인) JWT 인증 흐름은 다음과 같다.

1. 서버가 JWT를 생성하고 서버의 **개인키**로 JWT에 서명
`JWT = sign(header.payload, server_private_key)`
2. 클라이언트는 JWT를 저장해뒀다가 요청마다 JWT를 서버로 전송
3. 서버는 **서버의 공개키**로 서명을 검증
`verify(signature, header.payload, server_public_key)`
4. 검증이 성공하면 JWT를 정상 토큰으로 판단하고 인증을 허용.

이처럼 정상적인 구현에서는 **서버가 신뢰하는 공개키만 사용하여 서명을 검증**해야 한다.

하지만 취약한 일부 서버의 경우에는 다음과 같이 동작할 수 있다:  
`public_key = header.jwk`
`verify(signature, header.payload, public_key)`

즉, **JWT header**에 포함된 `jwk`값을 그대로 신뢰하여 검증 키로 사용하는 경우이다.  
이 경우 공격자는 서버가 사용할 검증 키를 직접 지정할 수 있게 된다.

예를들어 공격자가 **RSA key pair**를 생성 한 후,  
공격자의 **private key**로 JWT를 서명
`sign(header.payload, attacker_private_key)`

이후, JWT header의 `jwk`에 `attacker_public_key`를 삽입해서 서버에 전송하게 되면
취약한 서버는 **서버의 공개키**가 아닌, 공격자가 제공한 **공격자의 공개키**로 서명을 검증하게 된다.

결과적으로 서버는

> "서명이 올바르다 → 정상 토큰이다"

라고 판단하게 된다.

#### BurpSuite 예제

Lab: [**JWT authentication bypass via jwk header injection**]({% post_url 2026-03-08-jwk header injection-Lab %})

---

출처: [PortSwigger Academy](https://portswigger.net/web-security/jwt)
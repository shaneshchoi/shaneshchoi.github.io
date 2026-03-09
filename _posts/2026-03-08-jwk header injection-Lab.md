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

이번 실습은 BurpSuite를 사용하는 방법과 사용하지 않는 방법 두 가지 모두 알아볼 예정이다.  

---

## Solution
(coming soon)
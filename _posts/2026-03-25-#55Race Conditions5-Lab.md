---
title: Race Conditions 실습(5) - Exploiting time-sensitive vulnerabilities (PortSwigger Academy)
description: 시간 기반 token 생성 로직의 취약점을 이용해 다른 사용자 비밀번호를 탈취하는 공격
categories: [Web Security, Race Conditions, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Exploiting time-sensitive vulnerabilities

Lab Link: [Exploiting time-sensitive vulnerabilities](https://portswigger.net/web-security/race-conditions/lab-race-conditions-exploiting-time-sensitive-vulnerabilities)

---

## 핵심 포인트

이 lab은 전형적인 race condition이 아니라  
**시간 기반(token generation) 로직이 깨져있는 문제**다.
password reset token이 완전히 랜덤이 아니며, timestamp 기반으로 생성됐다.
결과적으로, 서로 다른 유저라도 **같은 시간에 요청하면 동일 token 생성이 가능 하다.**

즉, **"같은 타이밍 + 다른 username" 조합으로 token을 재사용**할 수 있다.

---

## 취약점 개념

비밀번호 재설정 흐름은 보통 이렇게 동작한다.

1. reset 요청
2. token 생성
3. email로 링크 전송
4. token 검증 후 password 변경

여기서 중요한건 token 생성 방식이 안전해야 한다는 점이다.

이 lab에서는 token이 다음 특징을 가진다:

- 길이가 항상 동일
- 요청할 때마다 값이 바뀜
- 하지만 완전히 랜덤은 아님

---

## 공격 흐름

### 1. reset token 동작 확인

사진(0)
우선 로그인 페이지에서 **Forgot password**를 클릭 해준다.
사진(1)
주어진 사용자 `wiener`로 reset link를 전송 해주자.

이후에 email 서버에 접속하면 password reset link가 도착한것을 확인할 수 있다.
사진(2)

password를 새로 변경 해주준다.
사진(3)

정상적인 흐름대로 진행 했으니, 요청을 분석 해보자.

### 2. 요청 분석

사진(4)

`POST /forgot-password`요청을 보면 `csrf`와 `username`파라미터를 받는다.  
우리의 목표인 `carlos` username을 입력해서 race condition을 이용하면 `carlos`의 계정 비밀번호 링크 또한 우리에게 전송이 될 수도 있다.

따라서 같은 요청 두개를 그룹화 해주고, 하나의 username을 `carlos`로 설정했다.

사진(5)

> Password Reset 그룹 (parallel)

사진(6)

> 타겟 username 설정

이메일 서버에 접속하면 링크가 도착한 것을 확인할 수 있다.

사진(7)

### 3. 문제점

해당 링크에는 `wiener`의 username은 보이지만 함께 요청한 `carlos`는 보이지 않는다.  
단순히 타이밍이 안맞았다기에는 여러번 반복해도 같은 결과가 나왔다.

따라서 응답 속도를 비교 해보았다.

사진(8)

> `wiener` 요청 응답 속도

사진(9)

> `carlos` 요청 응답 속도

응답시간에 약 300ms차이를 보인다. 이게 문제 일 수도 있어 임의의 요청을 맨 앞에 더해
Warming request를 추가해주었으나, 결과는 마찬가지였다.

다른 요소가 문제되는것 같아 찾아보던 중, `Cookie: phpsessionid=` 라는 부분이 눈에 들어온다.

사진(10)

같은 세션을 공유하고 있기 때문에 실패하는 것일 수도 있어, 해당 쿠키를 삭제한 후에 새로운 session cookie를 받아서 둘중 하나의 요청에 넣어줬다.

이후에 다시 전송을 하니 `"Invalid CSRF token"`에러가 발생했다. 단순히 Session Cookie만 변경해서 검증이 되지 않았다.

처음부터 다시 password reset을 보내고 다른 csrf토큰을 획득해서 request에 삽입해주고 시도했다.

메일 서버에 들어가보니 여전히 `carlos`이름이 포함된 링크는 포함되어있지 않았다.
사진(11)

이번 랩의 목적이 time-sensitive 취약점을 exploit하는것이기 때문에 해당 토큰이 단순히 timestamp를 기반으로 만들어졌을 확률이 있어  
`wiener`에서 `carlos`로 username만 바꿔서 url에 접속 해보았다.
```
https://0aed00ad041442e381ca481d00ca0084.web-security-academy.net/forgot-password?user=carlos&token=df6579d353922d348919fea9bf6daba9b99d94b9
```

이때 비밀번호를 reset할수있는 창이 새로 나왔고, 성공적으로 `carlos`의 비밀번호를 변경할 수 있었다.

사진(12)
이후에 변경된 비밀번호로 로그인을 해주면

사진(13)

`carlos`계정으로 로그인되었고 Admin Panel에 접속이 가능한것을 확인할 수 있었다.

**Admin Panel**에 접속해 우리의 목표인 `carlos`를 삭제하면 랩이 완료된다.

---

여담이지만, 성공한 두 요청 역시 response time갭이 300ms정도 존재했는데,  
알고보니 여기서 요청의 차이는 크게 중요하지 않다는것을 배웠다. 
```
1단계 - 요청
t = 0ms     요청 A 도착
t = 5ms     요청 B 도착

2단계 - token 생성
t = 10ms    A token 생성
t = 11ms    B token 생성  ← 같은 timestamp

3단계 - 응답 
t = 200ms   B 응답 완료
t = 500ms   A 응답 완료
```

이미 2단계 - token 생성단계에서 같은 timestamp를 기반으로 토큰을 서버에서 제작했기 때문에,  
응답시간과 상관없이 단순히 username만 변경하는것으로 탈취할 수 있었다.





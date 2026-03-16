---
title: Authentication Vulnerabilities (PortSwigger Academy) - 기타 인증 메커니즘 취약점
description: Remember-me 기능과 Password Reset/Change 과정에서 발생할 수 있는 인증 취약점 정리
categories: [Web Security, Authentication Vulnerabilities]
tags: [웹 보안, Authentication Vulnerabilities]
toc: true
---

# Authentication Vulnerabilities - Other authentication mechanisms

로그인 페이지 외에도 대부분의 웹사이트에는 **계정 관리 기능**이 존재한다.

대표적인 기능으로는

- 비밀번호 변경
- 비밀번호 재설정
- 로그인 유지(Remember me)

같은 것들이 있다.

문제는 이런 기능들이 로그인 페이지만큼 철저하게 검증되지 않는 경우가 많다는 점이다.  
특히 공격자가 **자신의 계정을 만들 수 있는 환경**이라면 상황이 훨씬 쉬워진다. 직접 계정을 생성하고 각 기능의 동작 방식을 충분히 관찰할 수 있기 때문이다.

---

## Keeping users logged in

많은 사이트는 브라우저를 닫아도 로그인 상태가 유지되도록 **Remember me** 기능을 제공한다.

일반적으로 로그인 시 **remember-me 토큰**을 생성하고 이 값을 **persistent cookie**에 저장한다. 이후 요청에서 이 쿠키를 확인해 로그인 상태를 유지한다.

이 쿠키만 있으면 로그인 과정을 건너뛸 수 있다.  
그래서 remember-me 토큰은 **예측이 불가능한 값**이어야 한다.

하지만 일부 웹사이트는 토큰을 단순한 방식으로 만든다.

```
username + timestamp
```

또는

```
username + password
```

이런 구조는 공격자가 분석하기 쉽다.  
자신의 계정을 만들 수 있다면 자신의 쿠키 값을 관찰하면서 **토큰 생성 규칙을 유추할 수 있다.**

구조가 보이면 공격자는 다른 사용자 계정을 대상으로 **remember-me 쿠키를 brute-force**하는 시도를 할 수 있다.

### 단순 인코딩의 문제

일부 사이트는 쿠키 값을 **Base64 같은 방식으로 인코딩**한다.

하지만 Base64는 암호화가 아니라 단순 인코딩이다.  
값을 복원하는 데 특별한 장벽이 없다.

해시를 사용하더라도 **salt가 없다면** 공격 난이도는 크게 높아지지 않는다.  
공격자가 해시 알고리즘을 알아내면 wordlist를 해싱해 비교하는 방식으로 공격할 수 있다.

이 방식은 특히 **로그인 시도 제한을 우회하는 공격**으로 이어질 수 있다.  
로그인 요청에는 제한이 있어도 **cookie 추측에는 제한이 없는 경우**가 있기 때문이다.

#### BurpSuite 예제

Lab: [**Brute-forcing a stay-logged-in cookie**]({% post_url 2026-03-14-#24Stay-logged-in cookie-Lab %})

---

### Cookie에서 password 정보 노출

드물지만 쿠키 안에 **password 관련 정보**가 포함되는 경우도 있다.

예를 들어 이런 형태다.

```
username : password hash
```

이 경우 공격자는 쿠키 값을 디코딩하거나 분석하여 **사용자의 password hash를 직접 확보**할 수 있다.

공격자가 반드시 자신의 계정을 만들 필요는 없다.

- XSS 취약점을 이용한 cookie 탈취
- 네트워크 공격을 통한 세션 가로채기
- 브라우저 악성 스크립트 같은 방법 으로 다른 사용자의 remember-me 쿠키를 탈취할 수도 있다.

쿠키를 확보한 뒤 공격자는 해당 값의 구조를 분석하여 **쿠키 생성 방식을 역추적**할 수 있다.

특히 웹 애플리케이션이 **잘 알려진 오픈소스 프레임워크**를 사용하고 있다면  
remember-me 쿠키 생성 방식이 **공개 문서에 이미 설명되어 있는 경우도 있다.**

---

### Hash를 이용한 password 추측

쿠키 안에 password hash가 포함되어 있다면 공격자는 이를 이용해 **offline password cracking 공격**을 시도할 수 있다.

공격자는 먼저 쿠키에서 얻은 hash 값을 분석한다.  
이때 가장 간단한 방법은 **hash 값을 그대로 검색해보는 것**이다.

이미 인터넷에는 수많은 password 리스트에 대한 **hash 데이터베이스**가 공개되어 있기 때문에,  
사용자의 password가 흔한 단어라면 단순히 hash 값을 검색하는 것만으로도  
원래 password를 알아낼 수 있는 경우가 있다.

검색만으로 결과가 나오지 않는다면 공격자는 다음 단계로 넘어간다.  
password wordlist를 이용해 **offline cracking**을 수행하는 것이다.

예를 들어 `rockyou` 같은 유명한 password 리스트를 사용하면  
수백만 개의 password 후보에 대해 hash 값을 계산한 뒤  
쿠키에서 얻은 hash와 일치하는 값을 찾을 수 있다.

특히 hash에 **salt가 적용되어 있지 않은 경우**,  
공격은 매우 빠르게 실행 가능하며, 이런 Brute-force 공격에 특히 취약하다.

#### BurpSuite 예제

Lab: [**Offline password cracking**]({% post_url 2026-03-14-#25Offline password cracking-Lab %})

---

## Resetting user passwords

사용자가 비밀번호를 잊어버리는 상황을 대비해 대부분의 웹사이트는 **password reset 기능**을 제공한다.

이 기능은 일반 로그인과 달리 **기존 인증 정보를 확인할 수 없는 상태**에서 동작한다.  
그래서 시스템은 이메일이나 reset 링크 같은 **다른 방식으로 사용자를 확인**한다.

이 과정이 제대로 설계되지 않으면 **계정 탈취로 이어질 수 있다.**

### Sending passwords by email

보안이 제대로 구현된 시스템이라면 **사용자의 기존 비밀번호를 그대로 보내는 일은 발생하지 않는다.**

비밀번호는 보통 **해시 형태로 저장**되기 때문이다.

그래서 일부 시스템은 새로운 비밀번호를 생성해 이메일로 전달하기도 한다.

하지만 이메일은 안전한 채널로 보기 어렵다.

- 메일함은 장기간 보관된다
- 여러 기기에서 동기화된다
- 네트워크 환경에 따라 중간자 공격이 가능하다

그래서 이메일로 전달된 비밀번호는 **짧은 시간 안에 만료되거나 바로 변경하도록 설계하는 것이 안전하다.**

### Resetting passwords using a URL

좀 더 안전한 방식은 **password reset URL**을 사용하는 것이다.

하지만 URL 구조가 단순하면 문제가 된다.

```http
http://vulnerable-website.com/reset-password?user=victim-user
```

이 구조에서는 공격자가 `user` 값을 다른 username으로 바꿔  
다른 계정의 reset 페이지에 접근할 수 있다.

### 안전한 구현 방식

더 안전한 방식은 **추측하기 어려운 토큰 기반 reset 링크**다.

```http
http://vulnerable-website.com/reset-password?token=a0ba0d1cb3b63d13822572fcff1a241895d893f659164d4cc550b421ebdd48a8
```

이 경우 서버는 토큰을 확인해 어떤 계정의 비밀번호를 재설정하는지 판단한다.

또한 토큰은

- 일정 시간이 지나면 만료
- reset 완료 후 즉시 폐기

되는 것이 일반적이다.

하지만 일부 시스템에서는 **reset form 제출 시 토큰 검증을 다시 하지 않는 경우**가 있다.

이 경우 공격자는 자신의 reset 페이지를 이용해  
**다른 사용자 계정의 비밀번호를 변경할 수 있다.**

#### BurpSuite 예제

Lab: [**Password reset broken logic**]({% post_url 2026-03-15-#26Password reset logic-Lab %})

### Password reset poisoning

reset 링크가 동적으로 생성되는 경우 **password reset poisoning** 공격이 가능하다.

공격자는 reset 요청을 조작해 **reset URL의 host 값을 변경**할 수 있다.

이 링크가 피해자에게 전달되면 피해자는 공격자가 만든 서버를 통해 reset 요청을 진행하게 된다.

그 결과 공격자는 **reset token을 탈취**할 수 있다.

#### BurpSuite 예제

Lab: [**Password reset poisoning via middleware**]({% post_url 2026-03-15-#27Password reset poisoning-Lab %})

---

## Changing user passwords

비밀번호 변경 기능은 보통

- 현재 비밀번호 입력
- 새 비밀번호 입력
- 새 비밀번호 확인

단계를 거친다.

이 과정은 로그인과 같은 검증 로직을 사용하기 때문에 **동일한 취약점이 발생할 수 있다.**

특히 password change 페이지가 **로그인 없이 접근 가능하다면 위험하다.**

예를 들어 username 값이 hidden field로 전달되는 구조라면  
공격자가 요청을 수정해 **다른 사용자 계정을 대상으로 요청을 보낼 수 있다.**

이 과정에서

- username enumeration
- password brute-force

같은 공격이 가능해질 수 있다.

#### BurpSuite 예제

Lab: [**Password brute-force via password change**]({% post_url 2026-03-15-#28Password change brute-force-Lab %})

---

출처: [PortSwigger Academy](https://portswigger.net/web-security/authentication/other-mechanisms)
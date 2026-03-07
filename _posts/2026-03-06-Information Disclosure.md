---
title: Information Disclosure
description: 웹 애플리케이션에서 발생하는 정보 노출 취약점 정리
categories: [Web Security]
tags: [information disclosure]
toc: true
---

# Information Disclosure

Information disclosure(정보 노출)는  
웹 애플리케이션이 **의도하지 않게 민감한 정보를 사용자에게 노출하는 취약점**을 의미한다.

이 취약점은 단순히 데이터 유출 문제로 끝나는 경우도 있지만,  
더 심각한 공격의 **출발점이 되는 경우가 많다.**

예를 들어 서버의 디렉토리 구조, 사용 중인 프레임워크 버전,  
또는 내부 API 경로 같은 정보가 노출되면  
공격자는 이를 바탕으로 추가 공격을 설계할 수 있다.

---

## 왜 발생하는가 (Root Cause)

Information disclosure 취약점은 다양한 이유로 발생하지만  
대부분 다음 세 가지 범주로 정리할 수 있다.

### 내부 정보가 제거되지 않은 채 배포되는 경우

개발 과정에서 남겨둔 정보가  
운영 환경에서도 그대로 노출되는 경우가 있다.

예를 들면 다음과 같다.

- HTML 주석에 남겨진 개발자 메모
- 테스트용 API 경로
- 임시 파일

이 정보들은 화면에는 보이지 않지만  
브라우저 개발자 도구나 Burp Suite를 통해 쉽게 확인할 수 있다.

---

### 잘못된 서버 설정

서버 설정이 적절하게 이루어지지 않은 경우에도  
정보 노출이 발생할 수 있다.

대표적인 예는 다음과 같다.

- 디버그 모드 활성화
- 과도하게 상세한 에러 메시지
- Directory listing 활성화

이러한 설정은 개발 환경에서는 유용하지만  
운영 환경에서는 공격자에게 중요한 단서를 제공할 수 있다.

---

### 애플리케이션 설계 문제

애플리케이션의 동작 방식 자체가  
정보를 노출하도록 설계된 경우도 있다.

예를 들어 서로 다른 오류 상황에서  
서로 다른 응답을 반환하는 경우  
공격자는 이를 이용해 내부 데이터를 추측할 수 있다.

이러한 문제는 다음 공격과 자주 연결된다.

- Username enumeration
- SQL Injection
- Access control bypass

---

## Impact

Information disclosure의 영향은  
노출된 정보의 종류에 따라 크게 달라진다.

### 사용자 데이터 노출

예를 들어 다음 정보가 유출될 수 있다.

- 사용자 이름
- 이메일 주소
- 금융 정보

이 경우 직접적인 데이터 유출 사고로 이어질 수 있다.

---

### 기술 정보 노출

다음과 같은 정보도 자주 노출된다.

- 사용 중인 프레임워크
- 서버 버전
- 디렉토리 구조

이 정보 자체는 큰 문제가 아닐 수도 있지만  
다른 공격을 위한 단서로 사용될 수 있다.

예를 들어 특정 프레임워크의 버전이 노출되면  
공격자는 해당 버전에 존재하는 공개 취약점을 찾아 공격할 수 있다.

---

## Information Disclosure가 발생하는 대표적인 위치

정보 노출은 웹 애플리케이션의 다양한 위치에서 발견된다.

### Files for web crawlers

웹 크롤러에게 탐색하지 말아야 할 경로를 알려주는 파일이다.

예:
```text
/robots.txt
/sitemap.xml
```

이 파일에는 다음과 같은 정보가 포함될 수 있다.

- 관리자 페이지 경로
- 비공개 디렉토리
- 테스트 페이지

공격자는 이를 통해 숨겨진 경로를 발견할 수 있다.

---

### Directory listing

웹 서버 설정에 따라  
디렉토리의 파일 목록이 그대로 노출될 수 있다.

예:
```text
/backup/
/logs/
/temp/
```

이 경우 공격자는 민감한 파일을 쉽게 발견할 수 있다.

---

### 개발자 코멘트 (Developer comments)

HTML 코드 안에 남겨진 개발자 주석도  
정보 노출의 원인이 될 수 있다.

예:

```html
<!-- TODO: remove admin endpoint before production -->
```

---

### Error messages

가장 흔한 정보 노출 원인 중 하나는
과도하게 상세한 에러 메시지다.

에러 메시지에는 다음과 같은 정보가 포함될 수 있다.

- 데이터베이스 종류
- 서버 경로
- 프레임워크 이름
- 버전 정보

이 정보는 공격자가 환경을 파악하는 데 큰 도움이 된다.

예:
`/product?productId=1` 이라는 요청을
`/product?productId=abc` 로 임의로 에러를 발생시킬때

![설명](/assets/burp_images/information-disclosure/error_message1.png){:width="1200px"}
> 요청

![설명](/assets/burp_images/information-disclosure/error_message2.png){:width="1200px"}
> 응답

**Apache Struts 2 2.3.31** 버전을 사용하는걸 확인 할 수 있다.

---

### 디버깅 데이터 (Debugging Data)

디버그 데이터가 운영 환경에서 노출 될 수도 있다.

예:

- 세션 변수 값
- 내부 서버 주소
- 암호화 키
- 백엔드 Credentials

공격자 입장에서 매우 가치 있는 정보다.

예:
![설명](/assets/burp_images/information-disclosure/error_message3.png){:width="1200px"}
> 요청

![설명](/assets/burp_images/information-disclosure/error_message4.png){:width="1200px"}
> 응답

주석에 숨겨져있는 `/cgi-bin/pipinfo.php`를 통해 민감한 정보 접근 가능

---

### User account pages

사용자 프로필에는 다양한 개인 정보가 포함된다.

예: `/user/profile`

일반적으로 사용자는 자신의 정보만 볼 수 있어야 하지만  
로직 오류가 있을 경우 다른 사용자의 정보에 접근할 수 있다.

예: `GET /user/personal-info?user=shane`

이 경우, IDOR 취약점과 연결될 수 있다.

---

### 백업파일 소스코드 노출

개발 과정에서 생성된 백업 파일이 서버에 남아있을 수 있다.

예:
```text
index.php~
config.php.bak
```

이런 파일을 요청하면 소스 코드가 그대로 노출될 수 있다.

예시:

`/robots.txt` 접속,  
```text
User-agent: *
Disallow: /backup
```
을 반환.

공격자는 `/backup` 에 접속,  

![설명](/assets/burp_images/information-disclosure/error_message5.png){:width="1200px"}

`backup/ProductTemplate.java.bak` 파일 오픈

![설명](/assets/burp_images/information-disclosure/error_message6.png){:width="1200px"}

Hard coded된 Postgres DB 비밀번호 탈취

---

### 안전하지 않은 설정 (Insecure configuration)

웹사이트는 잘못된 설정 으로 인해 정보 노출 취약점이 발생할 수 있다.  
이는 다양한 서드파티 기술의 설정을 정확히 이해하지 못한 채 사용하거나, 개발 환경에서 사용하던 디버깅 기능을 운영 환경에서 비활성화하지 않았을 때 자주 발생한다.

예를 들어 HTTP `TRACE` 메서드는 디버깅 목적으로 설계된 기능으로, 요청을 그대로 응답에 반환한다. 이 기능이 활성화되어 있을 경우 reverse proxy가 추가한 내부 인증 헤더 등 민감한 정보가 노출될 수 있다.

예시:

![설명](/assets/burp_images/information-disclosure/error_message7.png){:width="1200px"}

`TRACE /admin` 접속 후에 응답을 살펴보면 

**x-custom-ip-Authorization** 이라는 헤더를 자동으로 추가하는걸 볼 수 있음

이 헤더를 이용해서 localhost가 접근하는것 으로 위장할 수 있다.
`X-custom-IP-Authorization: 127.0.0.1`

---

### 버전 컨트롤 (Version Control History)

일부 웹사이트는 `/.git` 디렉토리를 공개하는데,  
이 경우 공격자는 Git 기록을 다운로드해 코드 변경 내역을 분석할 수 있다.

---

이어서 정보 노출 취약점 테스트 과정에 대해 다뤄보겠다.
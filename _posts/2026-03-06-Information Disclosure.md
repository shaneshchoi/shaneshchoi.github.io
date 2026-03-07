---
title: Information Disclosure
description: 웹 애플리케이션에서 발생하는 정보 노출 취약점 정리
categories: [Web Security, Information Disclosure]
tags: [웹보안, 기타]
toc: true
---

# Information Disclosure 이란?

Information disclosure(정보 노출)은  
웹 애플리케이션이 **의도하지 않게 민감한 정보를 사용자에게 노출하는 취약점**을 의미한다.

이 취약점은 단순히 데이터 유출 문제로 끝나는 경우도 있지만,  
더 심각한 공격의 **출발점이 되는 경우가 많다.**

예를 들어 서버의 디렉토리 구조, 사용 중인 프레임워크 버전, 또는 내부 API 경로 같은 정보가 노출되면 공격자는 이를 바탕으로 추가 공격을 설계할 수 있다.

---

## 왜 발생하는가? (Root Cause)

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

## 방어 방법

Information disclosure는 발생할 수 있는 경우의 수가 매우 다양하기 때문에  
단 하나의 방법으로 완전히 막기는 어렵다.

하지만 개발 과정과 운영 환경에서 몇 가지 기본 원칙을 지키는 것만으로도  
정보 노출 가능성을 상당히 줄일 수 있다.

---

### 민감한 정보에 대한 기준 정의

정보 노출 문제를 예방하기 위해서는  
먼저 어떤 정보가 외부에 공개되면 안 되는지 명확히 정의하는 것이 필요하다.

개발 과정에서는 내부 경로나 프레임워크 버전 같은 정보가  
큰 문제가 되지 않는다고 생각하기 쉽다.  

하지만 공격자의 입장에서는 이러한 정보들이  
애플리케이션 구조를 이해하는 중요한 단서가 될 수 있다.

따라서 조직 내부에서 어떤 정보가 민감한 정보에 해당하는지 기준을 정하고  
개발자들이 이를 인지하도록 하는 과정이 중요하다.

---

### 코드 감사 (Code Auditing)

Information disclosure는 종종 개발 과정에서 남겨진 코드나 주석 때문에 발생한다.

개발자가 디버깅을 위해 남겨 둔 내용이  
배포 이후에도 그대로 남아 있는 경우가 대표적인 사례다.

예를 들어 다음과 같은 HTML 주석은  
애플리케이션의 내부 구조를 추측할 수 있는 단서를 제공할 수 있다.

```html
<!-- TODO: remove admin endpoint before production -->
```

이러한 정보는 사용자에게 보이지 않더라도
브라우저 개발자 도구나 Burp Suite를 통해 쉽게 확인할 수 있다.

따라서 QA 단계나 빌드 과정에서
불필요한 주석이나 디버그 코드가 남아 있는지 확인하는 과정이 필요하다.

---

### Generic Error Message 사용

에러 메시지는 정보 노출이 발생하기 쉬운 부분 중 하나다.

개발 환경에서는 문제를 빠르게 확인하기 위해
데이터베이스 오류나 내부 경로를 그대로 출력하는 경우가 많다.

예를 들어 다음과 같은 메시지는
공격자에게 많은 정보를 제공할 수 있다.

```text
SQL error near column password_hash in table users
```

이 메시지를 통해 공격자는 다음과 같은 정보를 추측할 수 있다.

- 데이터베이스 사용 여부
- 테이블 이름
- 컬럼 이름

따라서 사용자에게는 가능한 한 단순한 오류 ex. `invalid request` 와 같은 메시지만 제공하는 것이 좋다.  
자세한 오류 정보는 서버 로그에만 기록하고 사용자에게 직접 노출되지 않도록 설계해야 한다.

---

### Debug 기능 비활성화

개발 환경에서는 디버그 기능이 문제 해결에 큰 도움이 된다.

하지만 운영 환경에서 이러한 기능이 활성화되어 있다면
애플리케이션 내부 정보가 그대로 노출될 수 있다.

예를 들어 디버그 페이지에는
다음과 같은 정보가 포함되는 경우가 많다.

서버 파일 경로

내부 변수 값

데이터베이스 연결 정보

대표적인 예로 스택 트레이스가 사용자에게 그대로 출력되는 경우가 있다

```text
Exception in thread "main" java.sql.SQLException
at com.example.database.UserRepository.findUser(UserRepository.java:42)
```

이러한 정보는 공격자가 애플리케이션 구조를 이해하는 데 도움을 줄 수 있다. 

따라서 운영 환경에서는 반드시 디버그 모드와 스택 트레이스 출력이 비활성화되어 있는지 확인해야 한다.

---

### 서버 및 프레임워크 설정 점검

정보 노출은 애플리케이션 코드뿐 아니라
서버 설정 문제로 인해 발생하는 경우도 많다.

대표적인 사례가 디렉토리 목록 노출(directory listing)이다.

예를 들어 웹 서버 설정이 잘못된 경우
다음과 같이 서버 내부 파일 목록이 그대로 노출될 수 있다.

```text
Index of /backup

config.php.bak
database.sql
old_config.zip
```

이러한 파일들은 원래 외부에 공개될 의도가 없었더라도
공격자가 쉽게 접근할 수 있게 된다.

또한 외부 프레임워크나 라이브러리를 사용할 때는
기본 설정이 보안에 적절한지 확인하는 과정이 필요하다.

사용하지 않는 기능이나 옵션은 비활성화하는 것이
정보 노출 위험을 줄이는 데 도움이 된다.


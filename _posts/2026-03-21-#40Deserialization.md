---
title: Insecure Deserialization (PortSwigger Aademy) - 개념과 기본 원리 (Part 1)
description: 직렬화와 역직렬화 개념, 그리고 Insecure Deserialization이 왜 위험한지 간단한 예시와 함께 정리
categories: [Web Security, Deserialization]
tags: [웹 보안, Deserialization]
toc: true
---

# Insecure Deserialization - 개념과 기본 원리

웹 애플리케이션은 종종 객체 형태의 데이터를 그대로 다루지 못하고,  
저장하거나 전송할 수 있는 형태로 바꿔서 사용한다.

예를 들면 이런 경우다.

- 세션 정보를 쿠키나 서버 저장소에 보관
- API 요청이나 응답으로 객체 전달
- 파일, 캐시, DB 등에 객체 상태 저장

이때 객체를 문자열이나 바이트 형태로 바꾸는 과정을 **serialization** 이라고 하고,  
반대로 그 데이터를 다시 객체로 복원하는 과정을 **deserialization** 이라고 한다.

이 구조 자체는 흔하고 정상적인 기능이다.  
문제는 **사용자가 조작할 수 있는 데이터**를 서버가 그대로 deserialization 할 때 생긴다.

이번 글에서는 개념을 짧게 정리하고,  
다음 글에서 실제로 serialized object를 수정하면서 어떻게 exploit 되는지 이어서 보도록 하겠다.

---

## Serialization / Deserialization

먼저 간단한 예시부터 보자.

서버에 이런 객체가 있다고 해보자.

    User {
      username: "shane",
      isAdmin: false
    }

이 객체를 그대로 전송할 수 없기 때문에,  
이런 형태로 바꿔서 사용한다.

    {"username":"shane","isAdmin":false}

이게 serialization이다.

그리고 이 데이터를 다시 받아서

    User {
      username: "shane",
      isAdmin: false
    }

이렇게 복원하는 것이 deserialization이다.

여기서 중요한 점은  
값만 저장되는 것이 아니라 **객체의 상태 전체가 유지된다는 것**이다.

---

## 문제 발생

위 예시의 구조 자체는 제가 없으나,  
서버가 사용자가 보낸 데이터를 그대로 객체로 복원할 때 문제가 생길 수 있다.

예를 들어 이런 데이터를 전달받는다고 가정해보자.

    {"username":"shane","isAdmin":false}

이 값을 사용자가 바꿀 수 있다면

    {"username":"shane","isAdmin":true}

`"isAdmin":true` 로 설정해 보내는 것도 가능하다.

서버가 이걸 그대로 객체로 복원하면  
관리자 권한을 가진 객체가 만들어질 수 있다.

또한, 객체는 단순한 데이터 묶음이 아니라

- 어떤 클래스인지
- 어떤 속성을 가지는지
- 어떤 메서드와 연결되는지

이런 정보까지 같이 묶여 있다.

그래서 공격자는 값만 바꾸는 것이 아니라  
**객체 자체를 바꾸는 방식**으로도 접근할 수 있다.

이런 이유로 Insecure Deserialization은  
종종 **Object Injection**이라고도 불린다.

---

## 애플리케이션 기능과 연결되는 경우

예를 하나 보자.

사용자가 탈퇴하면 프로필 이미지를 삭제하는 기능이 있다고 가정한다.

    unlink($user->image_location);

정상적인 경우라면 문제 없다.

그런데 `image_location` 값이  
사용자가 조작한 serialized 데이터에서 온다면 이야기가 달라진다.

    image_location = "/etc/passwd"

이렇게 바뀐 상태에서 탈퇴 기능이 실행되면  
서버는 그 경로를 그대로 사용하게 되고,  
결과적으로 서버가 `/etc/passwd`를 삭제하려고 시도한다.

`unlink()`와 같은 기능의 문제가 아니라, 그 기능에 들어가는 데이터가 조작됐으며 검증을 하지 않는다는 점에 있다.

---

## 취약한 이유

이 취약점은 단순히 입력값 검증 문제처럼 보일 수 있다.

하지만 실제로는 구조적인 문제에 가깝다.

개발자가 deserialization 이후에 검증하려고 해도,  
객체는 복원되는 순간부터 이미 애플리케이션 동작에 영향을 줄 수 있다.

또 현대 웹 애플리케이션은 프레임워크와 라이브러리를 많이 사용한다.  
그 안에는 다양한 클래스와 메서드가 들어 있고,  
공격자는 이 중에서 예상하지 못한 동작을 찾아 활용할 수 있다.

그래서 이 취약점은 단순히 입력값 검증보다는
연계되는 동작 내부에 취약한 포인트가 있는 경우 타겟이 될 수 있다.

상황에 따라

- 권한 상승
- 인증 우회
- 파일 접근 또는 삭제
- 서비스 장애
- 원격 코드 실행

같은 문제로 이어질 수 있다

모든 역직렬화 취약점이 바로 코드 실행으로 이어지는 것은 아니지만,  
구조 자체가 위험하기 때문에 영향이 커지기 쉽다.

---

## 정리

Insecure Deserialization은  
사용자가 조작할 수 있는 데이터를 서버가 그대로 객체로 복원할 때 발생한다.

겉으로 보기에는 단순한 데이터 처리처럼 보이지만,  
실제로는 서버 내부 객체와 기능 흐름에 직접 영향을 줄 수 있다.

처음에는 추상적으로 느껴질 수 있는데,  
이 취약점은 예시를 직접 보면 훨씬 이해가 쉽다.

다음 글에서는 serialized 데이터를 실제로 식별하고,  
값과 타입을 직접 바꾸면서 어떻게 exploit 되는지 이어서 본다.

---

출처: [PortSwigger Academy](https://portswigger.net/web-security/deserialization)
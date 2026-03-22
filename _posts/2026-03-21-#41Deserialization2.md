---
title: Insecure Deserialization (PortSwigger Aademy) - 취약점 분석과 공격 (Part 2)
description: 애플리케이션 기능과 magic method를 이용해 역직렬화 취약점이 실제 공격으로 이어지는 과정 정리
categories: [Web Security, Deserialization]
tags: [웹 보안, Deserialization]
toc: true
---

# Insecure Deserialization - 실제 공격 흐름

앞에서는 serialized 데이터를 직접 수정하는 것까지 봤다.  
이번에는 한 단계 더 들어가서  
**애플리케이션 기능 + 객체 동작**이 어떻게 연결되는지 본다.

---

## 1. PHP 직렬화 포맷

PHP는 객체를 직렬화할 때 사람이 어느 정도 읽을 수 있는 문자열 형태로 바꾼다.

예를 들어 이런 객체가 있다고 하자.

```
$user->name = "carlos";
$user->isLoggedIn = true;
```

이걸 serialize하면 아래 처럼 표현될 수 있다.
```
O:4:"User":2:{s:4:"name":s:6:"carlos";s:10:"isLoggedIn":b:1;}
```

`O:4:"User"`
- O는 Object
- 4는 클래스 이름의 길이
- "User"는 클래스 이름

`:2:`
- 2는 이 객체가 가지고 있는 속성이 2개 라는 뜻이다.  
이 예제의 경우, `name`과 `isLoggedIn`이 되겠다.

`s:4:"name"`
- s는 string
- 4는 문자열 길이
- "name"은 속성명

뒤에 나오는 `s:6:"carlos"`는
첫번째 속성의 값이다.

두번째 속성도 동일하게 `isLoggedIn`이라는 속성명과 함께  
`b:0` 이라는 값이 나오는데, 이는 Boolean `false`를 뜻한다.
`b:1` 은 `true`라고 보면 된다.

공격자가 이 값을 임의로 `0`에서 `1`로 바꿔주면,
서버측에서 이걸 deserialize 하면서
`isLoggedIn`값이 `true`가 되는 객체를 만들게 된다.

만약 서버측에서 아래와 같이 설정되어 있다고 가정해보자.
```
$user = unserialize($_COOKIE);
if ($user->isLoggedIn === true) {
    // 자동 로그인 허용
}
```

쿠키 값을 그대로 믿고 `unserialize()`해서 객체를 만들고,  
그 객체의 속성을 곧바로 권한 체크에 사용한다.

서버는 전송받은 쿠키가 정상 객체라고 가정하기때문에 가능한 시나리오다.
하지만 공격자카 쿠키 안에 직렬화 데이터를 바꿔도 서버에서 객체를 복원하는 과정에서
로그인이 성공할 수 있는 것이다.

### BurpSuite 예제:

Lab1 : [Modifying serialized objects]({% post_url 2026-03-21-#42Deserialization1-Lab %})  

---

## 2. 데이터 타입 변경

앞서 본 예제는 값을 바꾸는 것으로 공격이 가능했다.
하지만 역직렬화(deserialization)에서는 값뿐만 아니라  
**데이터 타입**자체도 같이 바뀐다는 점도 중요하다.

### 문제 상황

예를 들어 다음 코드가 있다고 하자.

    $login = unserialize($_COOKIE);
    if ($login['password'] == $password) {
        // 로그인 성공
    }

여기서 연산자 `==`에는 하나의 문제가 있다.
이 연산자는 타입이 다르면 **자동으로 타입을 맞춰서 비교**한다.

PHP는 숫자와 문자열을 비교할 때 문자열을 숫자로 바꾼다.
예를 들어 아래와 같은 경우도 있지만

    5 == "5"
    → true

다른 경우에는 문자열이 숫자로 시작하면  
앞에 있는 숫자만 사용하는 경우도 있다.

    5 == "5abc"
    → 5 == 5
    → true

위 예제처럼 뒤에 있는 문자열은 그냥 무시된다.

그럼 `5 == "5abc"`예제에서 `5`가 빠진 `abc`만 들어가면 어떻게 될까?

PHP 7 기준으로 문자열이 숫자로 시작하지 않으면 `0`으로 처리된다.
예시로,

    0 == "hello"
    → 0 == 0
    → true

`hello`라는 문자열만 사용했을 뿐인데 `true`값이 나와버린다.

### 공격 흐름

공격자가 serialized object 안의 값을 이렇게 바꾼다고 하자.

    password = 0

여기서의 `0`은 문자열이 아니라 **정수 0**이다.

그리고 예를 들어 실제 비밀번호가 `shanesblog` 라면,  
이 경우 서버에서는 아래와 같이 비교 된다.

    0 == "shanesblog"
    → 0 == 0
    → true

그래서 비밀번호를 몰라도 로그인이 될 수 있다.

하지만 이 특수한 케이스는 실제 password가  
**숫자로 시작하지 않는 문자열**일 경우에만 가능하다.

동일하게 위 예제처럼 password = 0 으로 값을 변경하더라도,  
실제 비밀번호가 `123shane` 이라고 한다면

    0 == "123shane"
    → 0 == 123
    → false

이런 흐름으로 동작 한다.

또한 PHP 8부터는 

    0 == "shanesblog"

동일한 예제의 값이 `ture`가 아닌 `false`로 처리되기 때문에  
PHP 8에서는 이 공격이 막히게 된다. 하지만, 아래처러 **숫자로 시작하는** 케이스는 여전히 적용 가능하다.

    5 == "5abc"
    → true

#### BurpSuite 예제

Lab2 : [Modifying serialized data types]({% post_url 2026-03-21-#43Deserialization2-Lab %})  

---

## 3. Application 동작기능 활용

앞서 살펴본 예제들은 **isAdmim** 값을 바꾸거나  
**token** 을 바꾸는식으로 검증로직을 우회했다.

단순히 값과 데이터 타입을 바꾸는데에서 한 단계 더 나아가서  
객체가 어디에 사용되는지를 생각해볼 필요가 있다.

예시로, 사이트에 "회원 탈퇴시 프로필 이미지도 같이 삭제" 라는 기능이 있다고 가정해보자.  
코드는 대충 이런 느낌이다.
```
unlink($user->image_location);
```
정상적인 경우
```
/var/www/images/profile_image.jpg
```
위 경로를 삭제한다.

### 문제 발생

여기서의 `image_location`이 serialized object에서 왔다고 한다면  
사용자가 조작 가능하다.

Payload를 `image_location = "etc/passwd"`로 설정후에 탈퇴 기능 실행을 하게 되면  
서버는 아래 코드를 실행한다. 
```
unlink("/etc/passwd")
```
즉, 프로필 이미지를 삭제하려는 원래 의도와는 다르게  
시스템 파일 삭제 시도를 하는 결과가 나오게 된다.

#### BurpSuite 예제

Lab3 : [Using application functionality to exploit insecure deserialization]({% post_url 2026-03-21-#44Deserialization3-Lab %})  

---

## 4. Magic Method

**Magic Method**란 특정 상황이 되면 자동으로 실행되는 함수를 말한다.

대표적인 예로, PHP 기준
- `__construct()` - 객체 생설될 때 실행
- `__wakeup()` - unserialize 될 때 실행
- `__destruct()` - 객체가 사라질 때 실행

우리가 앞서 살펴본 것 처럼 무언가 수정을 하거나,  
직접 기능을 실행할 필요 없이 역직렬화(deserialize)를 할때 실행시킬 수 있다는 점이 특징이다.

### 공격 흐름 - PHP 

예를 들어 이런 클래스가 있다고 보자.

```php
class User {
  public $file;

  function __wakeup() {
    unlink($this->file);
  }
}
```
이 경우, 객체가 역직렬화 되면 자동으로 `unlink()`기능이 실행된다.

공격자는 위에 `unlink()`함수 내부로 지정되는 `file`에 대한 path를 서버에 넣는다.
```
O:4:"User":1:{s:4:"file";s:23:"/home/carlos/morale.txt";}
```
이렇게 되면, 서버가 역직렬화를 실행하고 `__wakeup()`을 실행,  
결과적으로 서버로 보냈던 `/home/carlos/morale.txt`파일이 삭제된다.

이전 예제들과의 차이는
```
POST /delete-account
```
이런 요청을 보내야 unlink 실행됐으나

Magic method 방식은
```
GET /my-account (아무 요청)
Cookie: (조작된 serialized object)
```
위와 같은 요청을 보내도 서버가 내부적으로 deserialize를 하는 순간 실행이 된다는 차이점이 있다.

Java에서는 `readObject()`가 비슷한 역할을 한다.

```java
private void readObject(ObjectInputStream in) {
  // 여기 코드 자동 실행됨
}
```
마찬가지로 역직렬화 되는 순간 실행되게 된다.

서버가 만약 클래스 검증을 안하게 된다면, 
아무 클래스나 넣어도 객체로 만들어 주기 때문에 공격이 성공할 수 있다.

순서로 보면 아래와 같다.
1. 서버에 어떤 클래스들이 있는지 살펴본다.
2. 그 중에 취약한 코드가 있는 클래스를 찾는다.
3. 해당 클래스로 serialized object를 만든다.
4. 서버에 전달한다.

예제와 함께 알아보자.

#### BurpSuite 예제

Lab4 : [Arbitrary object injection in PHP]({% post_url 2026-03-21-#45Deserialization4-Lab %})  

---

## 5. Gadget chain

실제 서비스에서는 `unlink`, `exec`같은 코드가 노출되어 있는 경우가 드물다.
**Gadget**은 이미 서버 코드 안에 존재하는 코드 조각으로,  
- 값을 복사하는 함수
- 로그 찍는 함수
- 문자열 처리하는 함수
등과 같이 별것 아닌것처럼 보여지는 코드 조각들이다.

하지만 이런 조그만 코드 조각들도 이어 붙이게 되면 문제가 될 수도 있다.

개념적으로 보면 여러 개의 정상 코드를 연결해서 공격을 만든다고 이해하면 된다.

1. Magic method 실행됨 (시작점)
2. A 함수 호출
3. A 함수 결과가 B 함수로 넘어감
4. B 함수 결과가 C 함수로 넘어감
5. 마지막에 취약한 함수를 실행

간단한 예시:

~~~php
class A {
  public $data;

  function __destruct() {
    B::process($this->data);
  }
}

class B {
  static function process($input) {
    C::run($input);
  }
}

class C {
  static function run($cmd) {
    system($cmd);
  }
}
~~~

각각 나눠보면
- A → 그냥 데이터 전달
- B → 그냥 중간 처리
- C → system 실행 이지만,

이걸 연결 해보면 결국 system()까지 도달해서 흐름을 유도하는 공격이라고 보면 된다.
체인의 시작점 (Kick-off gadget)은 우리가 이전에 살펴봤던 Magic method:  
- `__wakeup()`
- `__destruct()`
- `readObject()` 
등이 보통이다.

이게 실행되면서 첫 번째 gadget이 호출된다.
gadget chain은 여러 클래스와 여러 함수, 그리고 전체적인 흐름의 이해가 필요하기 때문에,  
소스코드 없으면 성공하기 힘들다.

그 다음에 등장하는게 **pre-built gadget chain**이다.

현실에서는 많은 서비스가 **동일한 프레임 워크**와 **동일한 라이브러리**를 사용한다.
예로 `Apache Commons`, `Symfony`, `Laravel`등이 있다.

그래서 한 번 발견된 chain은 다른 취약한 사이트에서도 재사용이 가능하다.
**ysoserial**은 관련된 payload를 자동으로 만들어주는 도구다.

### Ysoserial

**ysoserial**의 사용 흐름은 다음과 같다:
1. 서버가 사용하는 라이브러리 추정
2. ysoserial에서 해당 chain을 선택
3. 실행할 명령어 (command)를 넣음
4. Serialized object (payload) 생성
5. 서버에 전달

결론적으로 gadget chain자체로 취약하다기 보다는,  
이전 예제들과 마찬가지로 untrusted data를 역직렬화 하는 것이 취약점으로 작용된다.

ysoserial Github 링크: [ysoserial git respository](https://github.com/frohoff/ysoserial)

#### BurpSuite 예제

Lab5 : [Exploiting Java deserialization with Apache Commons]({% post_url 2026-03-21-#46Deserialization5-Lab %})  

---

## 6. Gadget chain2

Insecure deserialization에서 우리의 목표는 보통 두 단계로 나눠져있다.

**1단계: 정말 서버가 역직렬화를 하는지의 여부**  
- 쿠키나 파라미터에 직렬화 객체가 보이기는 함
- 근데 서버가 그걸 읽고 복원(readObject)하는지는 확신이 없음

**2단계: Exploit 가능 여부**  
- 라이브러리 맞는 gadget chain 찾기
- RCE까지 연결

바로 **RCE Payload**를 넣게되면 실패의 원인이 너무 많다.

예를들어,
- gadget이 안 맞을 수도 있고
- 라이브러리가 없을 수도 있고
- outbound traffic이 막혀 있을 수도 있고
- 서버가 예외만 내고 끝날 수도 있다.

위의 BurpSuite Lab5 예제에서도 봤듯 Apache Commons Collection을 사용한다는 점을 일러주지 않았다면 상당히 오랜 노가다가 필요했을 것이다.

그래서 일단 **deserialize 자체가 발생하는지 탐지용 gadget으로 확인**하는 과정을 거친다.

이때 쓰는 대표적인게 `URLDNS`와 `JRMPClient`이다.

### 방법1. URLDNS chain

가장 범용적인 탐지용 gadget이다. 
Payload 안에 특정 도메인(URL)을 넣으면 서버가 그 객체를 deserialize하는 순간 **DNS lookup**이 발생한다.

예를들어 Burp Collaborator 주소가:
```
shane123.burpcollaborator.net
```
이라고 하면,  
URLDNS payload는 "이 객체를 deserialize 하면 `shane123.burpcollaborator.net`에 대해 DNS 조회를 하게 만들어라"와 같은 의미다.
실무에서는 *Burp Collaborator*를 사용해서 DNS 요청이 들어오는지 확인 가능하다.

#### URLDNS chain 장점

이 방법은 **특정 라이브러리**를 필요로 하지 않는다.  
일반적인 RCE Gadget은 `CommonsCollections`, `BeanUtils`, `Spring`, `Groovy`같이 특정 라이브러리가 서버에 있어야 한다.  
근데 URLDNS는 그런 복잡한 gadget chain보다는 단순해서 거의 모든 Java환경에서 탐지용으로 사용 가능하다.

또한 임의 코드 실행이 아니라 "상호작용"만 확인하면 된다.  
우리가 원하는건 명령어 실행이나 파일 삭제같은 RCE가 아니라,  
"서버가 내 Payload를 실제로 역직렬화 했는가"다.

서버가 DNS 요청만 보내줘도, "아, 역직렬화가 일어났구나"를 알 수 있다.

#### URLDNS chain의 한계

이 방법은 네트워크가 막혀있거나, 서버가 외부 DNS를 못 나가면 안 보일 수 있다.  
또한 deserialize는 일어났지만 outbound DNS가 차단되면 흔적을 찾을 수도 없다.

이 방법의 대안이 `JRMPClient`이다.

### 방법2. JRMPClient chain

이건 DNS 대신 **TCP 연결시도**를 이용하는 탐지용 chain이다.  
Payload안에 IP주소를 넣으면, 서버가 deserialize하는 순간 그 IP로 TCP Connection을 시도한다.
→내 payload가 역직렬화가 되면, 서버가 저 IP로 접속 하려고 할 것이다.    
DNS가 막혀있는 환경에서도 사용 가능하며, raw IP를 사용해야 한다.

JRMPClient는 hostname대신 IP주소를 요구한다. 이유는  
DNS 해석 단계를 거치지 않고 바로 해당 IP로 소켓 연결을 시도하게 하려는 것.

즉, URLDNS처럼 "DNS query를 보겠다"가 아니라,  
실제 TCP 연결 시도를 해보겠다는 접근이다.

때문에 DNS가 막힌 환경에서도 서버는 여전히 외부 IP로 TCP연결을 시도할 수 있다.
만약에 외부는 방화벽으로 막혀 있다고 하더라도, 막히는 과정 자체가 응답 시간 차이로 드러나게 된다.

#### 탐지 방법 (시간차 탐지)

1. 두 개의 payload를 준비한다 (내부망 또는 가까운 IP, 외부 차단된 IP)
2. 두 개의 payload를 모두 테스트 해서 결과를 비교한다.

각각의 Payload를 A와 B라고 했을때
내부 IP와 외부 차단된 IP 두 개의 payload를 모두 보냈을때 만약 역직렬화가 발생한다면,  
A 요청은 빠르게 응답하게 되며,  
B 요청은 지연(Timeout) 발생을 하게 된다.

응답 시간에 차이가 있다는건 Deserialize가 발생한다는 의미이다.
이는 특히 outbound가 차단되거나, DNS가 안나가거나, 에러가 보이지 않을때 사용 가능하다.

### 방법3. PHPGGC (PHP 용)

PHPGGC는 PHP용 Gadget chain 모음이다. 

- Laravel
- Synfony
- WordPress

같은 프레임워크 기반 chain을 제공한다.  
PHP판 ysoserial과 비슷한 개념이라고 볼 수 있겠다.

PHP도 insecure deserialization이 자주 나오는데

예를들어:
- `unserialize()` 사용
- magic method (`__wakeup`, `__destruct`, `__toString`) 존재
- framework/library 내부 gadget chain 존재
이런 상황이면 PHP에서도 gadget chain으로 exploit이 가능하다.

이럴때 PHP 환경에서는 ysoserial 대신 PHPGGC를 쓴다.

### Java vs PHP 및 실전 흐름 정리

정리하자면

#### 1. Java
- 직렬화 포맷: Java serialized object
- 도구: ysoserial
- gadget 예: CommonsCollections, URLDNS, JRMPClient

#### 2. PHP
- 직렬화 포맷: PHP serialized string
- 도구: PHPGGC
- gadget 예: Laravel, Monolog, Symfony 등 프레임워크 기반 chain

#### 실전 흐름

Java deserialization이 의심되는 상황에서 보통 이렇게 간다.

**1단계**

serialized object처럼 보이는 값 찾기  
예: `rO0AB...`

**2단계**

탐지용 gadget 사용  
예: `URLDNS`, `JRMPClient`

**3단계**

deserialize 발생 여부 확인  
- Collaborator DNS hit
- TCP connection
- 응답 시간 차이

**4단계**

그 다음에 RCE gadget 시도  
예: `CommonsCollections4`

이전 실습에서 2,3단계인 deserialize탐지 단계가 추가 되었다.

### BurpSuite 예제

Lab:

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

출처: [PortSwigger Academy](https://portswigger.net/web-security/deserialization/exploiting)
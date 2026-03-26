---
title: Insecure Deserialization 실습(5) - ysoserial을 이용한 Java gadget chain 공격 (PortSwigger Academy)
description: ysoserial을 사용하여 Java deserialization 취약점을 통해 명령어 실행까지 이어지는 과정 정리
categories: [Web Security, Deserialization, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Exploiting Java deserialization with Apache Commons

Lab Link: [Exploiting Java deserialization with Apache Commons](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-java-deserialization-with-apache-commons)

---

## 핵심 포인트

이 lab은 이미 만들어진 gadget chain을 사용한다.  
ysoserial 툴로 payload 생성해서 exploit 해볼 예정이다.

앞서, 이번 실습에는 **Apache Commons Collections Library**를 사용하며,  
최종 목표는 carlos의 home directory에 있는 `morale.txt`를 삭제하는 것 이다.

---

## 실습 흐름

### 1. 로그인 및 데이터 확인

로그인 후 요청을 보면 session cookie에 들어가있는 값이 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-0.png" alt="설명" width="800">
</div>

해당 session cookie를 Base64 decode해보면 `ac ed 00 05`로 시작하는걸 확인 할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-1.png" alt="설명" width="500">
</div>

Java Object를 serialize 하면 일반적으로 `AC ED 00 05`로 시작하며,  
이 값을 **magic byte**라고 한다. 

따라서 해당 값이 확인되면 Java serialized object일 가능성이 높다고 판단할 수 있다.

다른 비슷한 패턴들도 있으니 참고하면 좋을 것 같다.

| 시작값     | 의미                 |
| ------- | ------------------ |
| `rO0AB` | Java serialization |
| `eyJ`   | JSON   |
| `PHN2Z` | XML                |
| `UEsDB` | ZIP / JAR          |


마지막 부분이 `%3d%3d`로 `==` 문자가 URL Encoding이 된걸 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-2.png" alt="설명" width="400">
</div>

따라서 전체적인 Java Object 구조는,
```
Serialize (binary)
→ Base64 Encode
→ URL Encode
→ Cookie 
```
순서대로 만들어졌다고 알 수 있다.

Serialize (직렬화) 순서를 고려했을때, 역직렬화를 하게된다면 서버에서 Cookie를 만들었던 반대 순서대로 되돌린다는것도 알 수 있다.

**역직렬화 과정:**
```
Cookie
→ URL decode
→ Base64 Decode
→  Deserialize
```
---

### 2. 공격 방향

서버에서 deserialize 발생  
→ gadget chain 가능  
→ command 실행 가능

순서대로 exploit 할 예정이다. 

우선, BurpSuite Extension인 **Java Deserialization Scanner**를 먼저 돌려보자.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-3.png" alt="설명" width="800">
</div>

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-4.png" alt="설명" width="500">
</div>
Session Cookie를 Intertion Point로 설정 해두고, Cookie를 직렬화 했던 순서 (Base64 Encoding → URL Encoding)  
순서대로 추가 한 후에 공격실행 해준다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-5.png" alt="설명" width="700">
</div>

결과를 보면 **Not Vulnerable**이라고 나온다.
정확한 결과를 확인하기 위해 *Configuration* 탭에서 `Verbose mode`를 활성화 시켜줬다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-6.png" alt="설명" width="500">
</div>

> Verbose 모드 활성화

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-7.png" alt="설명" width="700">
</div>

조금 더 자세한 결과값이 나온다.  
**Apache Commons Collections 3** - HTTP/2 500 Internal Server Error가 발생한다.

500 Internal Server 에러가 발생했다는 것은,  
Payload가 서버까지 도달해서 역직렬화 (Deserialization) 단계까지는 갔으나  
**gadget chain**, **타입 mismatch** 등으로 인해 발생했다는 뜻이다.

**Apache Commons Collections 4** 역시 마찬가지로 HTTP/2 500 Internal Server Error를 반환 했다.

내가 Java Deserialization Scanner를 잘못 사용하는건지는 몰라도,  
해당 Extension에서는 결과를 찾을 수 없어서 **ysoserial**로 Apache CommonsCollections에 대한 payload를 생성 해보겠다.

---

### 3. payload 생성

공격을 위해 다운받은 **ysoserial**을 준비 해주자.  
**ysoserial**은 Java gadget chain payload를 자동으로 만들어주는 툴이다.  
github 확인 해보면 아래와 같이 사용할 수 있다는 Documentation을 찾을 수 있다.  
링크: [ysosreial github](https://github.com/frohoff/ysoserial/blob/master/README.md)

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-8.png" alt="설명" width="800">
</div>

문제에서 `Apache Commons Collections Library`를 사용한다고 알려줬으니, 
우리의 타겟은 `CommonsCollections`가 되겠다.

Apache commons-collections 버전에 따라서 사용할 수 있는 payload가 각각 다르다.
해당 Documentation에 따르면,  
`Commons-Collections:3.1`은 **CommonsCollections1, 3, 5, 6, 7**을 사용해서 payload를 만들 수 있고,  
`Commons-Collections:4.1`은 **CommonsCollections2, 4**를 사용해서 만들 수 있다.

테스트를 위해 우선 **CommonsCollections1**을 사용해서 payload를 만들어보자.

`java -jar ysoserial-all.jar [payload] "[command]"`가 기본 명령어다. 

우리는 아래 명령어를 입력 해준다.
```
java -jar ysoserial-all.jar CommonsCollections1 "rm /home/carlos/morale.txt" | base64 -w 0 > output.txt
```

ysoserial의 출력은 기본 **raw binary**로 출력이 된다.  
예시) `\xac\xed\x00\x05sr...` 

앞서 살펴 봤듯, 우리가 Cookie값에 넣을수 있는 Session은 기본적으로 **Base64 Encoding**과 **URL Encoding**을 한 값이다.

따라서 `| base64`를 진행해서 binary → 문자열로 변환 시켜준다.

또한, 기본 Base64 동작은 76자마다 줄바꿈(\n)을 넣기때문에 HTTP Cookie에 줄바꿈이 적용되어 깨짐 현상이 생길 수 있다.  
그걸 방지하기 위해 `-w 0` 커맨드로 줄바꿈 없이 한 줄로 출력하게끔 만든다.

만들어진 `output.txt` 텍스트 파일을 확인하고 해당 값을 `Cookie: session=`헤더에 넣어준다.

그리고 만들어진 Cookie값을 URL Encoding 해준 후에 전송 해주면 된다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-9.png" alt="설명" width="800">
</div>


> 요청

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab5-10.png" alt="설명" width="700">
</div>

> 응답

Response를 살펴보면, HTTP/2 500 Internal Server Error를 반환하고 있으며,  
Lab화면에서도 아직 **Solved**화면이 나오지 않는것으로 봐서 `CommonsCollections1` payload는 실패 했다.

Apache Commons Collections 라이브러리 버전이 다른 것일수도 있어서,  
`CommonsCollections2`로 다시 진행 해줬더니 Lab이 해결 됐다.

서버는 여전히 HTTP/2 500 Internal Server Error를 반환하고있지만,  
우리가 보낸 `rm /carlos/home/morale.txt` RCE는 성공 했다. 

애플리케이션이 예상한 세션 객체를 복원하지 못하거나 예외를 처리하지 못한는 경우 500 Internal Server Error를 반환할 수 있다.
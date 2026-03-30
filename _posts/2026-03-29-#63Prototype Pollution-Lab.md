---
title: Prototype Pollution 실습(1) - DOM XSS via client-side prototype pollution (PortSwigger Academy)
description: query string를 통한 client-side prototype pollution으로 Object.prototype을 오염시키고, gadget을 이용해 DOM XSS를 발생시키는 과정
categories: [Web Security, Prototype Pollution, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: DOM XSS via client-side prototype pollution

Lab Link: [DOM XSS via client-side prototype pollution](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-dom-xss-via-client-side-prototype-pollution)

---

## 핵심 포인트

이 Lab은 **client-side prototype pollution**을 이용해  
브라우저에서 실행되는 JavaScript 로직을 오염시키고,  
그 결과 동적으로 생성되는 `<script>` 태그의 `src`를 공격자가 통제할 수 있게 되는 문제다.

최종적으로는 `data:` URL을 이용해 가장 기본적인 `alert(1)`을 실행시키면 된다.

이번 실습에서 핵심적으로 봐야 할 포인트는 두 가지다.  
첫 번째는 **query string**을 이용해 `Object.prototype`에 임의의 속성을 추가할 수 있는지,  
두 번째는 이렇게 추가된 속성이 실제 애플리케이션 코드에서 **gadget**으로 사용되는지이다.

---

## 개념 설명

### 1. Prototype pollution 이란?

자바스크립트 객체는 자신의 속성만 확인하는 것이 아니라, 필요한 경우 상위 prototype 체인을 따라 값을 탐색한다.

예를 들어:
```javascript
const obj = {};
console.log(obj.test);
```

원래 `test`는 존재하지 않기때문에 `undefined`가 나와야 하지만, 만약 누군가 **Object.prototype.test = "hello"**라고 만들어 놓았다면,  
`obj.test`는 **"hello"**를 반환할 수 있다.

즉, 개별 객체를 바꾸는 것이 아니라,  
**여러 객체가 공통으로 상속받는 prototype 자체를 오염**시키는 것이 prototype pollution이다.

---

### 2. Client-side prototype pollution

이번 랩에서는 **브라우저 안에서 실행되는 JavaScript** 취약점에 대해 다룬다.  
공격자는 URL 파라미터를 통해 값을 넣고, 프론트엔드 JavaScript가 해당 값을 적절히 검증하지 않은 채 처리하면서 브라우저의 `Object.prototype`이 오염된다.

그 결과 페이지 내 다른 객체들이 공격자가 심어놓은 속성을 마치 자신의 속성인 것처럼 상속받아 사용하게 된다.

---

### 3. Source, Gadget, 그리고 Sink

Prototype Pollution에서 핵심이 되는 개념은 Source, Gadget, 그리고 Sink이다.

- **Source**는 공격자가 값을 집어넣을 수 있는 시작점이다.  
예를들면 URL query, hash, JSON input같은 곳이다.  
Prototype Pollution에서는 `__proto__`같은 값을 여기로 넣어서 오염시킨다.

- **Gadget**은 오염된 값을 실제 동작에 연결해주는 중간 속성이나 코드이다.  
예를들어 앱이 `config.transport_url`같은 값을 읽는다면,  
이 `transport_url`이 **gadget**이 될 수 있다.

- 마지막으로 **Sink**는 최종적으로 위험한 동작이 일어나는 지점이다.  
예를들면 `innerHTML`, `eval()`, `script.src`같은 곳이다.  
오염된 값이 여기까지 들어가면 XSS같은 exploit이 발생하게 된다.

흐름으로 보면  
**Source**에서 값을 주입하고 → **Gadget**에서 해당 값을 읽은 뒤 → **Sink**에서 exploit이 발생한다.

---

### 4. Client-side prototype pollution 공격 흐름

순서는 다음과 같다.

1. URL 파라미터를 이용해서 `Object.prototype` 오염 가능 여부 확인
2. JS 코드 안에서 상속된 속성을 읽는 gadget 찾기
3. 그 gadget이 `<script src=...>`로 이어지는지 확인하기
4. `data:` URL을 이용해 JavaScript 실행
5. 목표인 `alert(1)`발생 시키기

---

## 공격 단계

### 1. Source 찾기

첫 번째 목표는 임의의 속성을 전역 `Object.prototype`에 추가할 수 있는 **source**를 찾는 것이다.

**source**는 위에서 설명 했듯, URL query, JSON input, hash등에서 추가가 가능한데,  
가장 먼저 URL query string을 통해 테스트해보자.  
```url
/?__proto__[shane]=blog
```

```url
https://0a51001b03b48a50809444ad0091009b.web-security-academy.net/?__proto__[shane]=blog
```

여기서 `__proto__`는 prototype 이며, `[shane]`은 테스트용 임의 속성, 그리고 `blog`는 그에 대한 값 이다.
이 값을 삽입했을 때 우리의 목표는  
```javascript
Object.prototype.shane= "blog"
```
와 같은 효과가 실제로 발생하는지 확인하는 것이다.

이처럼 임의의 속성과 값을 넣는 이유는,  
실제로 exploit을 하기 전에 오염이 가능한지 여부를 확인하는 목적으로 테스트한다.

---

### 1-2. DevTools에서 확인

브라우저 DevTools를 열고 Console 탭에서 `Object.prototype`을 확인해본다.

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab1-0.png" alt="설명" width="700">
</div>

사이트가 query string을 처리하는 과정에서 `__proto__`를 제대로 막지 못해  
`shane: "blog"`속성이 추가된 것을 확인할 수 있다.

여기까지 성공하면 해당 사이트는 **client-side prototype pollution source가 존재한다**고 확인이 된 것이다.

---

### 2. Gadget 찾기

다음 단계는 이 오염된 속성이 실제로 어디에서 사용되는지 확인하고, 이를 활용할 수 있는 gadget을 찾는 것이다.  
**DevTools**의 **Sources**탭으로 가서 페이지에 로드되는 **JavaScript**파일들을 살펴봤다.  

`deparam.js`와 `searchLogger.js`가 있는것을 확인할 수 있는데,  

`searchLogger.js`파일에서 아래와 같은 코드를 포함하고 있는 것을 확인할 수 있다.

```javascript
if(config.transport_url) {
    let script = document.createElement('script');
    script.src = config.transport_url;
    document.body.appendChild(script);
}
```

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab1-1.png" alt="설명" width="500">
</div>

`config.transport_url`이 존재하면 `<script>`태그의 `src`로 삽입시키는 코드이다.

원래라면 `transport_url`이 **config**에 정의되어있어야 하겠지만, 해당 코드를 참조해보면, 정의되어 있지 않다는 것을 확인할 수 있으며, `config.transport_url`을 읽을 때 해당 속성이 객체에 존재하지 않으면 prototype 체인을 따라 상위에서 값을 탐색하게 된다.

따라서 우리가 `Object.prototype.transport_url`을 오염시키면, `config.transport_url`이 그 값을 상속받게 될 수 있다.

여기서 `config.transport_url`이 **gadget**이 된다.

---

### 2-1. Gadget 테스트

이제 **Source**와 **Gadget**후보를 찾았으니 실제로 동작하는지 시험해볼 단계이다.

앞서 prototype pollution을 진행했던 것처럼 URL을 다음과 같이 변경 한다.
```
/?__proto__[transport_url]=shane
```

만약 성공적으로 pollution이 진행됐다면 우리가 기대하는 상태는 이런것이다:
```javascript
Object.prototype.transport_url = "shane"
```

확인을 위해 해당 URL을 입력 해준 후에, debugger 모드에서  
```javascript
script.src = config.transport_url;
```
코드가 어떤 값을 지니는지 확인 해봤다.

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab1-2.png" alt="설명" width="500">
</div>

성공적으로 **"shane"**을 반환하는것을 볼 수 있었다.

또한, HTML 페이지를 다시 렌더링 한 결과, 
```html
<script src="shane"></script>
```
가 반영되어있는것도 확인할 수 있었다.

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab1-3.png" alt="설명" width="600">
</div>

---

### 3. XSS payload 

**Source**와 **Gadget**이 검증이 되었으니 이제 이를 이용해 브라우저 측에서 스크립트를 실행시키면 된다.

우선 아래 `transport_url` 코드를 다시 확인 해보자.

```javascript
    if(config.transport_url) {
        let script = document.createElement('script');
        script.src = config.transport_url;
        document.body.appendChild(script);
    }
```

**script.src**는 코드가 아니라 보통 외부 스크립트 파일 URL을 받는다. 
예시:
```html
<script src="https://attacker-website.com/x.js"></script>
```

따라서 `"alert(1)'`만 삽입하게 되면, 아래와 같이 반환하며, 브라우저는 이를 `"alert(1)"`이라는 파일 경로로 해석하고 요청을 시도할 뿐이다.
```html
<script src="alert(1)"></script>
```

하지만 랩에서는 굳이 외부 서버를 만들지 않아도 `data:` URL을 사용할 수 있다.

Data URL의 기본 문법은 다음과 같다.
```
data:[MIME type],[data]
```

`,` 뒤부터 실제 데이터가 시작되며, 여기에 우리가 원하는 코드인 "alert(1);"을 삽입하면 된다.
브라우저는 이를 JavaScript 코드로 해석하여 `alert(1)`을 실행시킨다.

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab1-4.png" alt="설명" width="700">
</div>

> 브라우저에서 alert(1)이 실행된 모습

순서대로 다시 살펴보자면,  
1. query string이 파싱됨
2. `Object.prototype.transport_url = "data:,alert(1);"`발생
3. 앱이 `config.transport_url`참조
4. 상속된 값이 반환됨
5. HTML에 `<script src="data:,alert(1);">`생성
6. JavaScript 실행
7. `alert(1)`발생

---

## 정리

이번 실습에서는 사용자 입력을 기반으로 prototype 오염이 가능했고, 애플리케이션이 상속된 속성을 신뢰하고 사용했으며, 그 값이 DOM XSS sink까지 이어져 실제 실행으로 연결되었다.

근본적인 원인으로는

1. `__proto__`를 포함한 입력이 클라이언트측 로직에서 필터링 되지 않는 Prototype Pollution source가 존재했으며
2. `config.transport_url`처럼 원래 안전하다고 믿었던 속성이 prototype 체인에서 상속받아 `script.src` sink에 들어갔다는 점이다.
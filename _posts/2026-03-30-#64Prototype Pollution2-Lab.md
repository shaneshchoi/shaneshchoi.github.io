---
title: Prototype Pollution 실습(2) - DOM XSS via an alternative prototype pollution vector (PortSwigger Academy)
description: alternative prototype pollution vector를 이용해 Object.prototype을 오염시키고, eval gadget을 통해 DOM XSS를 발생시키는 과정
categories: [Web Security, Prototype Pollution, DOM XSS, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: DOM XSS via an alternative prototype pollution vector

Lab Link: [DOM XSS via an alternative prototype pollution vector](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-dom-xss-via-an-alternative-prototype-pollution-vector)

---

## 핵심 포인트와 취약점 개념

이번 Lab은 **Prototype Pollution + DOM XSS**를 결합한 문제이며,  
이전 실습과 동일하게 Prototype Pollution 취약점을 기반으로 하지만,  
이번 Lab에서는 이를 DOM XSS로 이어지게 만드는 과정에 초점을 둔다.
 

핵심 포인트는 다음과 같다.

- query string을 통한 client-side prototype pollution
- `__proto__`를 이용한 Object.prototype 오염
- eval() sink를 활용한 DOM XSS
- gadget property 조작을 통한 코드 실행

최종 목표는 마찬가지로 **alert() 실행**이다.

---

## 취약점 개념

Prototype Pollution은 공격자가 객체의 prototype을 조작하여  
애플리케이션 전반에 영향을 주는 취약점이다.

특히 JavaScript에서는 모든 객체가 `Object.prototype`을 상속받기 때문에,  
여기에 값을 주입하면 전역적으로 영향을 미칠 수 있다.

이때, polluted된 값을 특정 함수(eval, innerHTML 등)에 전달할 수 있다면  
**DOM XSS로 이어질 수 있다.**

---

## 공격 흐름

### 1. Prototype Pollution Source 찾기

먼저 query string을 통해 prototype을 오염시킬 수 있는지 확인한다.

```url
https://0afa0071031e31c480b4034400d8006a.web-security-academy.net/?__proto__[shane]=blog
```

이후 DevTools 콘솔에서 확인해보면 `shane`이 추가되지 않는것을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab2-0.png" alt="설명" width="700">
</div>

다른 방법으로 prototype 오염시킬수 있는 방법은 `[]`대신에 `.`을 사용할 수 있다.

```
https://0afa0071031e31c480b4034400d8006a.web-security-academy.net/?__proto__.shane=blog
```

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab2-1.png" alt="설명" width="700">
</div>

`shane: "blog"`가 반영 되었으며 Object.prototype이 성공적으로 오염된 것을 확인할 수 있다.

JavaScript에서는

```javascript
obj.a = 1
obj['a'] = 1
```

이론적으로 두 방식 모두 prototype에 접근할 수 있는 형태지만,  
실제 동작은 query parser의 구현에 따라 달라진다.  

특히 많은 라이브러리들은 `__proto__` 키를 필터링하기 때문에  
`__proto__[key]` 형태는 차단되는 경우가 많다.

```url
__proto__[shane]=blog
__proto__.shane=blog
```

이는, URL → JS객체로 변화되는 과정에서 결과가 다르게 반영될 수 있다.

예시로, 
```url
/?__proto__[shane]=blog
```

이건 그냥 문자열이 아니라  
내부적으로 아래와 같이 파싱된다:

```javascript
{
  "__proto__": {
    "shane": "blog"
  }
}
```

반면에 아래 url은 
```url
/?__proto__.foo.bar
```

이렇게 파싱 되거나  
```javascript
{
    "__proto__.shane": "blog"
}
```

또는 라이브러리에 따라
```javascript
obj.__proto__.shane = "blog"
```
경우에 따라서는 prototype에 직접 접근하는 형태로 처리될 수 있다.

---

### 2. `[]`와 `.`의 차이, 그리고 우회 가능한 이유

필터링이라기 보다는 라이브러리/코드 구현 방식의 차이가 있을 수 있다.

이는 `[]` 기반 파싱과 `.` 기반 파싱이 내부적으로 서로 다른 로직을 사용하기 때문이다.

일부 라이브러리는 `__proto__`를 key로 직접 사용할 경우 필터링하지만,  
`.`을 사용한 경우에는 단순 문자열 분할(split) 후 접근하기 때문에  
필터링을 우회할 수 있다.

대부분의 파서 (예: qs, jQuery, etc.)는 아래와 같이 `[]`로 처리되어있는건 nested object로 처리하려고 한다.
```
__proto__[shane]=blog
```

nested object로 처리하게 되면, 파서가 다음과 같이 해석을 하는데,
```javascript
obj["__proto__"]["shane"]= "blog"
```

여기서 `obj["__proto__"]`를 통해 Object.prototype에 접근해서 `Object.prototype.shane="blog"`와 같은 prototype pollution이 발생하는것이다.

근데 여기서 보안때문에 많은 라이브러리들이 prototype pollution방지를 위해 아래와 같이 막아둔다
```javascript
if (key === "__proto__") ignore()
```

결과적으로 `__proto__[shane]`이 무시되는것 이다.

반면에 dot notation 같은 경우에는 파서가 다르게 처리한다.
```
/?__proto__.shane=blog
```

```javascript
let key = "__proto__.shane".split('.')
// ["__proto__", "shane"]

obj[keys[0]][keys[1]] = "blog"
```

결과적으로 `Object.prototype.shane = "blog"`가 성공하게 된다.

---

### 3. Gadget 찾기

이제 prototype에 값을 주입할 수 있으므로, 이 값을 실제로 사용하는 코드 위치(gadget)와 코드를 실행시키는 지점(sink)을 찾아야 한다.

Source 탭에서 JS파일을 분석 해보면, `searchLoggerAlternative.js`라는 파일에

```javascript
eval('if(manager ... +manager.sequence+')
```
라는 코드를 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab2-2.png" alt="설명" width="700">
</div>

해당 코드는 `eval()`을 사용하며, `manager.sequence`값이 실행이 된다.

JavaScript의 `eval()` 함수는 문자열을 JavaScript 코드로 실행하는 함수이다.

따라서 공격자가 제어 가능한 값이 eval()로 전달될 경우,  
임의의 JavaScript 코드 실행으로 이어질 수 있다.

예를들어
```javascript
eval("alert(1)")
```
코드는 alert(1)을 실행하게 된다.

따라서 우리가 `manager.sequence`값에 임의의 스크립트 코드를 삽입하고, 이걸 `eval()`안에 넣을수 있게 된다면  
`eval()`함수가 우리의 코드를 대신 실행시켜준다. 따라서 여기서는 `eval()`이 sink가 된다.

`eval()`은 대표적인 sink이지만, 다른 sink도 존재한다.

| sink           | 위험      |
| -------------- | ------- |
| eval()         | 코드 실행   |
| innerHTML      | HTML 실행 |
| document.write | DOM 삽입  |

---

### 4. 디버깅 및 Exploit

URL 입력을 통해 `sequence`값을 **blog** 로 설정해놓고 디버깅을 진행 해보자.
```
https://0afa0071031e31c480b4034400d8006a.web-security-academy.net/?__proto__.sequence=blog
```

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab2-3.png" alt="설명" width="600">
</div>

prototype polluition은 성공적이었지만,  
우리가 설정한 **"blog"**와는 다르게 **"blog1"**이라는 값으로 나오는걸 볼 수 있다.

이는 Line 16에 설정되어 있는
```javascript
manager.sequence = a + 1;
```
때문이다. 

따라서 `alert(1)`를 프린트 하기 위해서는 뒤에 있는 1을 제거하는 방법을 강구해야한다.

```
https://0afa0071031e31c480b4034400d8006a.web-security-academy.net/?__proto__.sequence=alert(1);
```

우선 `sequence=alert(1)`으로 프로토타입을 오염시켜주고 디버거를 다시 진행한다.

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab2-4.png" alt="설명" width="600">
</div>

마찬가지로, `"alert(1);1"`이라는게 반환된다.

문자열 뒤에 숫자 `1`이 붙으면서 JavaScript 문법이 깨지기 때문에, 이를 보정하기 위해 trailing operator (`*`)를 추가한다.

<div class="img-left-block">
<img src="/assets/burp_images/Prototype-pollution/Prototype-pollution_Lab2-5.png" alt="설명" width="600">
</div>

이 외에도 '`-`', '`/`', '`||`' 등의 연산자도 사용 가능하다.
`alert(1)`이 실행되며 Lab이 해결된다.

---

## 정리

이번 Lab에서는 client-side Prototype Pollution과 DOM XSS가 결합된 취약점을 확인할 수 있었다.

- query string을 통해 Object.prototype을 오염시킬 수 있었으며
- `__proto__` 필터링이 존재했지만 dot notation을 이용해 우회할 수 있었다
- 오염된 값이 `eval()` 함수로 전달되면서 최종적으로 DOM XSS가 발생했다

이 취약점은 단순한 prototype 오염에서 끝나는 것이 아니라, 실제 코드 실행으로 이어질 수 있다는 점에서 매우 위험하다.

특히 `eval()`과 같은 sink와 결합될 경우 공격자가 원하는 JavaScript를 실행할 수 있기 때문에 치명적인 보안 문제로 이어질 수 있다.  
따라서 단순히 입력값 필터링이 아닌, 객체 구조 처리 방식과 코드 실행 지점까지 함께 고려한 방어가 필요하다.

따로 정리는 안했지만, `__proto__`를 bypass할 수 있는 방법으로는 `.constructor.prototype`을 사용하는 방법이 있다.

예를들어:

```javascript
__proto__.shane="blog"
constructor.prototype.shane="blog"
```
위 코드들은 `Object.prototype.shane = "blog`와 같은 코드들이다.

따라서 `constructor.prototype`으로 `__proto__`가 막히는 경우에 우회가 가능하다.
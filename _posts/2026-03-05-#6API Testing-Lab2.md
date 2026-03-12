---
title: API Testing 실습(2) - Finding and exploiting an unused API endpoint
description: 사용되지 않는 API endpoint를 이용해 가격을 조작해보자.
categories: [Web Security, API Testing, Lab]
tags: [웹 보안, 실습]
pin: false
math: true
mermaid: true
published: true
toc: true
---

# Lab: Finding and exploiting an unused API endpoint

Lab Link: [Finding and exploiting an unused API endpoint](https://portswigger.net/web-security/api-testing/lab-exploiting-unused-api-endpoint)

## Lab 개요

이 Lab은 숨겨진 API endpoint를 발견하고 이를 이용해 애플리케이션의 기능을 악용하는 과정을 다룬다.

> 목표: 숨겨진 API endpoint를 이용해 `Lightweight "l33t" Leather Jacket`을 무료로 구매

---

## Exploit 과정

상품 페이지에 접속하면 제품 정보를 불러오기 위해 API 요청이 발생한다.
Burp에서 HTTP history를 확인하면 다음과 같은 API 요청을 볼 수 있다.

`GET /api/products/1/price`

해당 요청을 Repeater로 보내 HTTP method를 변경해 보면서 어떤 기능이 허용되는지 확인한다.
먼저 `OPTIONS` 요청을 보내면 서버가 허용하는 HTTP method를 확인할 수 있다.

`OPTIONS /api/products/1/price`

![설명](/assets/burp_images/api-testing-lab/api_lab2-2.png){:width="1200px"}

응답을 보면 `GET` 뿐만 아니라 `PATCH` method도 허용되어 있는 것을 확인할 수 있다.

![설명](/assets/burp_images/api-testing-lab/api_lab2-3.png){:width="1200px"}
> 요청

![설명](/assets/burp_images/api-testing-lab/api_lab2-4.png){:width="1200px"}
> 응답

`PATCH` 요청을 보내보면 `Content-Type` 오류가 발생하며 서버는 `application/json` 형식을 요구한다.

헤더를 추가하고 요청을 다시 보내면 이번에는 `price` 파라미터가 필요하다는 에러 메시지가 반환된다.

![설명](/assets/burp_images/api-testing-lab/api_lab2-5.png){:width="1200px"}

이 에러 메시지를 기반으로 요청 body를 다음과 같이 구성한다.

``` text
Content-Type: application/json

{
"price": 0
}
```

요청이 정상적으로 처리되면 해당 상품의 가격이 `$0.00`으로 변경된다.

이후 상품 페이지를 새로고침하면 가격이 변경된 것을 확인할 수 있으며, 장바구니에 추가 후 주문을 진행하면 Lab이 해결된다.

---

## 취약점 분석

이 Lab의 핵심 취약점은 **사용되지 않거나 숨겨진 API endpoint가 적절한 접근 통제 없이 노출되어 있다는 점**이다.

RESTful API에서는 동일한 endpoint라도 HTTP method에 따라 서로 다른 기능이 동작할 수 있다.  
이번 Lab에서도 기본적으로는 `GET` 요청을 통해 가격 정보를 조회하도록 구현되어 있었지만, 동일한 endpoint에 `PATCH` 요청을 보내면 가격을 수정할 수 있는 기능이 존재했다.

문제는 이러한 기능이 사용자에게 노출되지 않은 내부 API였음에도 불구하고, 별도의 권한 검증 없이 접근이 가능했다는 점이다.

또한 `OPTIONS` 요청을 통해 서버가 허용하는 HTTP method를 쉽게 확인할 수 있었으며, 에러 메시지 역시 요청 형식을 구체적으로 알려주고 있어 공격자가 올바른 요청 구조를 빠르게 구성할 수 있었다.

결과적으로 공격자는 숨겨진 API 기능을 발견한 뒤 상품 가격을 `0`으로 변경하여 무료로 구매할 수 있었다.

---

## 방어 전략

사용되지 않거나 내부적으로만 사용되어야 하는 API endpoint는 외부에서 접근할 수 없도록 설계해야 한다.  
특히 개발 과정에서 테스트 목적으로 만들어진 endpoint가 Production 환경에 그대로 남아있는 경우 공격에 악용될 수 있다.

또한 API 요청에 대해 **적절한 authorization 검증**을 구현해야 한다. 가격 수정과 같은 민감한 기능은 일반 사용자에게 허용되어서는 안 되며, 관리자 권한이 있는 사용자만 수행할 수 있도록 제한해야 한다.

에러 메시지도 최소한의 정보만 제공하도록 구성하는 것이 중요하다. 요청 형식이나 필요한 파라미터를 상세하게 알려주는 메시지는 공격자가 올바른 요청을 구성하는 데 도움을 줄 수 있다.

마지막으로 API 설계 시 허용되는 HTTP method를 명확하게 제한하고, 필요하지 않은 method는 비활성화하는 것이 바람직하다.
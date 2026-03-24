---
title: Race Conditions 실습(1) - Limit overrun race conditions (PortSwigger Academy)
description: 병렬 요청을 이용해 discount code를 중복 적용하여 가격을 낮추는 race condition exploit
categories: [Web Security, Race Conditions, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: Limit overrun race conditions

Lab Link: [Limit overrun race conditions](https://portswigger.net/web-security/race-conditions/lab-race-conditions-limit-overrun)

---

## 핵심 포인트

이 lab은 **discount code 적용 로직에서 발생하는 race condition**을 이용한다.

- discount code는 **1회만 적용 가능**
- 하지만 **동시에 요청을 보내면 여러 번 적용 가능**
- 결과적으로 **상품 가격을 비정상적으로 낮출 수 있다**

---

## 취약점 개념과 목표

해당 기능의 내부 로직은 다음과 같이 동작한다.

1. discount code 사용 여부 확인  
2. 할인 적용  
3. DB에 사용 완료 기록  

우리의 목표는 이 사이에 존재하는 **race window**를 노린다.
여러 쿠폰 사용 요청이 동시에 들어오면  
모든 요청이 "아직 사용되지 않았다"라고 판단할 수 있다.

이 취약점을 이용해서 **Lightweight "l33t" Leather Jacket**을 보유한 돈 보다 더 싸게 구매 하는게 목표다.

---

## 공격 흐름

### 1. Application 동작 확인

로그인 후 상단을 보면 우리가 가진 **Store credit**과 사용할 수 있는 **PROMO20** 코드가 보인다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-0.png" alt="설명" width="700">
</div>

우리가 구매해야 할 **Light "l33t" Leather Jacket**을 카드에 넣으면 다음과 같이 보여진다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-1.png" alt="설명" width="700">
</div>

여기에 PROMO 코드를 적용하면 20% 할인이 적용된 것을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-2.png" alt="설명" width="700">
</div>

정상적으로는 1회만 적용되나,  
우리는 이 할인 코드 요청 여러개를 동시에 보내서 중복으로 할인을 받을 예정이다.

### 2. Burp 요청 전송

우선은 Burp Proxy Intercept를 켜주고 쿠폰을 적용시켜준다.  
그러면 Proxy를 가로챈 상태이기 때문에 아직 요청이 전송되지 않은 상태다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-3.png" alt="설명" width="700">
</div>

> Proxy Intercept 

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-4.png" alt="설명" width="700">
</div>

> 쿠폰 적용 Request

캡처한 `POST /cart/coupon` 요청을 Repeater로 전송해서 여러 개 복사한 뒤 Group 으로 묶어준다.
여러 요청을 거의 동시에 서버에 도달하도록 만들어주기 위해 `Send group in parallel (single-packet attack)`을 선택 해준다.  
이 기능은 race window를 맞추는 역할을 한다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-5.png" alt="설명" width="700">
</div>

요청을 전송한 뒤 Intercept를 해제하면  
Application에서 할인 코드가 여러 번 적용된 것을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-6.png" alt="설명" width="700">
</div>

여전히 할인률이 모자라지만 중요한 점은 공격이 성공했다는 점이다.  
더 많은 요청을 보내기 위해 다시 Proxy를 켜주고 Repeater 탭을 스무 개로 늘려준다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-7.png" alt="설명" width="700">
</div>

다시 요청 전송을 보내게 되면, 스무 개의 요청이 동시에 나가게 된다.
하지만, 응답을 확인해 보면 모든 요청이 동시에 도달하지 않았음을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-8.png" alt="설명" width="700">
</div>

> 성공 응답

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-9.png" alt="설명" width="700">
</div>

> 실패 응답

그럼에도 우리가 원하는 만큼의 할인은 적용이 됐다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab1-10.png" alt="설명" width="700">
</div>

따라서 이전 Race Conditions 개념글에서 다룬 내용중에 20-30개의 요청을 동시에 보냈을때 전부 성공하지 않으며 성공률을 높혀줄 뿐이라는 말이 와닿았다.
할인된 가격 **$37.62**에 결제 해주면서 Lab이 마무리 된다. 

---

## 정리:

Race condition은 타이밍이 취약점이다. 많이 보내는 것이 아니라 동시에 보내는 것이 핵심이나, 가능한한 많이 보내는 것이 성공률을 높힐수 있다는걸 이번 실습을 통해 배웠다. 
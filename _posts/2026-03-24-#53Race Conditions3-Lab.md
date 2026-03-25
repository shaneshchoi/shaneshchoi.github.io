---
title: Race Conditions 실습(3) - Multi-endpoint race conditions (PortSwigger Academy)
description: 서로 다른 endpoint를 동시에 호출하여 결제 검증과 주문 확정 사이의 race window를 악용하는 공격
categories: [Web Security, Race Conditions, Lab]
tags: [웹 보안, 실습]
toc: true
---

# Lab: Multi-endpoint race conditions

Lab Link: [Multi-endpoint race conditions](https://portswigger.net/web-security/race-conditions/lab-race-conditions-multi-endpoint)

---

## 핵심 포인트

이 lab은 **두 개의 서로 다른 endpoint를 동시에 호출**하여  
결제 로직의 race condition을 유발하는 문제다.

- `POST /cart` → 장바구니 수정  
- `POST /cart/checkout` → 결제 요청  

이 두 요청을 동시에 보내면  
**결제 검증 이후 주문 상태를 조작할 수 있다**

우리의 목표는 보유한 값보다 비싼 **Lightweight L33t Leather Jacket**을 구입하는것 이다.

---

## 취약점 개념

일반적인 결제 흐름은 다음과 같다.

1. 장바구니 확인  
2. 잔액 검증  
3. 결제 승인  
4. 주문 확정  

이 과정은 겉보기에는 하나의 요청처럼 보이지만,  
실제로는 여러 단계로 나뉘어 처리된다.

## 공격 흐름

### 1. Application 구조 파악

우선 Application이 어떻게 동작하는지 파악해볼 필요가 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-0.png" alt="설명" width="600">
</div>

주어진 `wiener:peter`계정으로 로그인해보면 **store credit**이 $100.00 있는것을 확인할 수 있다.  
이 금액을 가지고 $1330.00짜리 **Lightweight L33t Leather Jacket**을 구입 해야한다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-1.png" alt="설명" width="400">
</div>


우선은 가장 싸게 구입할 수 있는 **Gift card**를 구입 해보고 어떻게 동작 하는지 파악 해보자.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-2.png" alt="설명" width="600">
</div>

**Gift card**를 카트에 넣은 모습이다. 


여기서 **Place order**버튼을 누르면 주문이 되는것을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-3.png" alt="설명" width="400">
</div>


### 2. BurpSuite 요청과 응답 분석

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-4.png" alt="설명" width="800">
</div>

> `/cart` 요청

`POST /cart` 요청에는 우리가 담은 Gift card의 **productId**와 **Quantity**가 파라미터로 들어가있는것을 볼 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-5.png" alt="설명" width="800">
</div>

> `/cart/checkout` 요청

`POST /cart/checkout`요청은 카트에 담겨져 있는 제품을 결제 해주는 역할을 한다.

여기서 카트에 담긴 제품이 실제로 어디에 저장되는지 확인해 볼 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-6.png" alt="설명" width="700">
</div>

> `GET /cart` 요청에서 `session cookie`삭제후 요청

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-7.png" alt="설명" width="600">
</div>

> 응답

`GET /cart` 요청에서 `session cookie`를 제거했을 때 cart가 비어있는 것을 통해  
장바구니 상태가 client가 아닌 server-side session에 저장된다는 것을 확인할 수 있다.

따라서 모든 cart 관련 요청이 동일한 session을 기준으로 처리된다는 의미이며,  
여러 요청이 동일한 데이터를 동시에 수정할 수 있기 때문에  
race condition이 발생할 수 있는 조건을 충족한다.

그 다음은 응답 시간을 확인해 볼 수 있다.  

Burp Repeater에서 **Sequence - Single connection**을 사용해서 응답 시간을 정확하게 비교 해볼 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-8.png" alt="설명" width="700">
</div>

`POST /cart`와 `POST /cart/checkout`요청을 따로 보내게 됐을때는 각각 TCP Handshake, TLS Handshake, 네트워크 경로 등의 변수로  
endpoint때문인지 네트워크 connection문제인지 알 방법이 없다. 따라서 같은 connection에서 진행하는것으로 위 변수들을 제거 해주고, endpoint 처리 시간 차이만 남겨두기 때문에 endpoint timing을 비교하기에 좋은 방법이다.

요청을 보낸 결과, `POST /cart`는 **448 millisecond** 

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-9.png" alt="설명" width="400">
</div>

`POST /cart/checkout`은 **168 millisecond**를 반환 했다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-10.png" alt="설명" width="400">
</div>

두 요청간의 갭이 크기때문에 race condition이 아니라고 판단할 수 있지만,  
첫 요청은 endpoint처리와 연결 준비 시간이 합쳐지기 때문에 더 오래 걸리는 이유도 있다고 한다.
예를들어 반환된 **448 millisecond**에는
```
connection setup + TLS + 실제 처리 시간
``` 
등이 포함 되어있을 가능성이 크며,  
두 번째 요청에서 반환된 **168 millisecond**는 순수 endpoint처리 시간으로만 볼 수도 있다는 뜻이다.

만약 해당 딜레이가 첫 요청이기때문에 발생했다면, 우리는 첫 요청을 랜덤한 요청으로 먼저 보내고, 이후 위 두 요청을 각각 두번쨰, 세번째 요청으로 이어서 보낸다면, 두 요청 사이의 갭을 줄일 수 있을것이라고 가정할 수 있다.

이 개념을 **warming request**라고 하며,  
첫 요청에서 발생하는 connection 초기화 비용을 제거하기 위한 기법이다.

---

### 3. Parallel 요청과 Exploit

앞서 설명한 **warming request**를 Group tab 맨 앞에 위치를 시키고 아까와 동일하게 **Single connection**요청을 보낸다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-11.png" alt="설명" width="700">
</div>

각각 순서대로 아래와 같이 반환 하는걸 볼 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-12.png" alt="설명" width="400">
</div>

> 첫번째 요청 (Warming request) - *441 millisecond*

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-13.png" alt="설명" width="400">
</div>

> 두번째 요청 (`POST /cart`) - *148 millisecond*

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-14.png" alt="설명" width="400">
</div>

> 세번째 요청 (`POST /cart/checkout`) - *140 millisecond*

이로써 첫 번째 응답만 connection setup등의 네트워크 문제로 딜레이가 생겼으며, 이후에 요청된 `POST /cart`와 `/POST /cart/checkout`요청에 대해서는 갭이 현저히 줄어든 것을 확인할 수 있었다 → endpoint가 유사한 타이밍으로 처리됨을 확인.

즉, 두 요청이 동일한 시점에 실행될 가능성이 존재하며  
race window에 진입할 수 있는 조건이 갖춰진다.

이제 공격을 위해 우선은 카트에 현재 보유한 돈보다 적은 $10짜리 Gift Card 물건을 넣어준다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-15.png" alt="설명" width="600">
</div>

이후에 `POST /cart`요청에서 `ProductId`를 우리의 타겟인 **Lightweight L33t Leather Jacket**의 품번인 `1`로 변경해주고  
Race condition exploit을 위한 parallel reqeust(동시 요청)로 변경 해준다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-16.png" alt="설명" width="800">
</div>

이후 여러 번 요청을 반복하면, 두 요청이 race window에 정확히 들어가는 순간이 발생하며 공격에 성공한다.  
보유하고 있는 돈과 상관 없이 해당 품목이 구입 된것을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab3-17.png" alt="설명" width="600">
</div>

## 정리:

이 취약점은 checkout 과정이 하나의 동작이 아니라 여러 단계로 나뉘어 처리된다는 점에서 발생한다.

공격자는 다음과 같은 방식으로 race condition을 유발할 수 있다:

1. 카트에는 $10짜리 Gift Card가 존재하는 상태
2. `POST /cart` 요청으로 고가의 상품을 추가
3. 동시에 `POST /cart/checkout` 요청을 전송

서버 내부에서는 다음과 같은 흐름이 발생한다.

1. checkout 시작
2. 기존 cart 상태 ($10)를 기준으로 금액 검증 수행
3. 잔액 검증 통과
4. 그 사이에 cart가 $1340으로 변경됨
5. 주문 확정 단계에서 현재 cart 상태를 다시 참조
6. 최종 주문에는 $1340이 반영됨

즉, 검증은 이전 상태 ($10)를 기준으로 수행되고,  
주문 생성은 변경된 상태 ($1340)로 이어지는 구조다.

이 랩을 하면서도 처음에는 checkout 요청이 들어오는 순간 cart가 스냅샷으로 고정되어 처리될 것이라고 생각했기 때문에, 이 취약점이 직관적으로 와닿지는 않았다.

하지만 실제로는 검증과 주문 확정이 분리되어 있고, 그 사이에 shared state가 변경될 수 있다는 점에서 race condition의 본질을 이해할 수 있는 실습이었다.
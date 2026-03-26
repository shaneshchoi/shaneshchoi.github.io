---
title: Race Conditions 실습(4) - Single-endpoint race conditions (PortSwigger Academy)
description: 하나의 endpoint에 병렬 요청을 보내 email 변경 로직의 race condition을 유발하는 공격
categories: [Web Security, Race Conditions, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Single-endpoint race conditions

Lab Link: [Single-endpoint race conditions](https://portswigger.net/web-security/race-conditions/lab-race-conditions-single-endpoint)

---

## 핵심 포인트와 취약점 개념

이 lab은 **하나의 endpoint에 서로 다른 요청을 동시에 보내는 방식**으로  
email 변경 로직의 race condition을 유발하는 문제다.

email 변경 기능은 다음과 같은 흐름을 가진다.

1. 새로운 email 입력  
2. 해당 email로 confirmation 메일 전송  
3. 링크 클릭 시 변경 확정  

이 과정은 하나의 동작처럼 보이지만, 실제로는:

- email 저장  
- 메일 전송 작업 등록  
- 템플릿 렌더링  

등 여러 단계로 나뉘어 처리된며, 이 사이에 race window가 존재한다.

---

## 공격 목표

`carlos@ginandjuice.shop` 이메일은 **admin 권한 초대 상태**라는 정보가 주어졌다.
우리의 목표는 해당 이메일을 탈취함으로써 admin 권한 획득을 목표로 한다.

---

## 공격 흐름

### 1. 동작 확인

주어진 계정 `wiener:peter`로 로그인 한 뒤에 
임의의 email로 변경 요청을 진행 한다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-0.png" alt="설명" width="600">
</div>


email 서버를 확인 해보면, email 변경 확인 링크가 도착한것을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-1.png" alt="설명" width="800">
</div>

해당 링크를 누르면 My Account에서 이메일이 성공적으로 바뀌는것을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-2.png" alt="설명" width="600">
</div>

---

### 2. Burp 분석

앞서 보냈던 **Update email**요청을 분석해보자.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-3.png" alt="설명" width="600">
</div>

요청을 보면 `email`과 `csrf`파라미터를 사용하는 것을 확인할 수 있다.

해당 요청을 동시에 여러개 보냈을 때의 동작을 확인하기 위해,  
Repeater에서 동일한 요청을 10개로 복사한 뒤 그룹화 시켜준다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-4.png" alt="설명" width="700">
</div>

각 요청마다 email 값을 다르게 설정 해주자.  
`email=abc{$요청번호}@exploit...server.net`

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-5.png" alt="설명" width="700">
</div>

만약 10개의 요청을 동시에 보냈을때 순서가 뒤바뀌거나 중복되는 요청이 있다면 race condition이 존재한다고 가설을 세울 수 있다.
그룹화 된 요청을 **Parallel(동시 요청)**로 전송해준다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-6.png" alt="설명" width="500">
</div>

> 결과

요성 순서와 상관없이, 서로 다른 email 값들이 뒤섞여 전송된 것을 확인할 수 있다.

이는 다음과 같은 구조를 의미한다:
- email 전송 작업이 먼저 시작되고
- 이후 DB에서 email 값을 읽어 메일 템플릿을 생성하는 과정에서
- 다른 요청이 DB 값을 덮어씀

그 결과, 잘못된 email로 메일이 전송되는 상황이 발생한다.

특히 `abc2@exploit...server.net`이 여러 번 반복 되는것을 확인할 수 있는데,  
이는 여러 요청이 값을 덮어쓰는 과정에서 특정 시점에 남아 있던 값이  
여러 메일 생성 과정에서 반복적으로 사용된 것이다.

따라서, DB값을 나중에 다시 읽고 있으며 shared state를 쓰고 있다는 뜻이 된다.

---

### 3. Exploit

Race condition이 존재함을 확인했으므로,  
이제 **admin** 권한을 가진 `carlos@ginandjuice.shop`을 이용해 exploit을 진행한다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-7.png" alt="설명" width="700">
</div>

두 개의 요청을 동시에 전송한다. 

- 요청 1 → `abc1@exploit...server.net`  
- 요청 2 → `carlos@ginandjuice.shop`

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-8.png" alt="설명" width="700">
</div>

---

### 4. 결과

이메일 서버를 확인 해보면, `carlos@ginandjuice.shop`로 변경할 수 있는 confirmation 메일이 도착한 것을 확인할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-9.png" alt="설명" width="700">
</div>

해당 confirmation을 클릭하면 내 계정(공격자 계정)이메일이 `carlos@ginandjuice.shop`으로 변경되며,  
admin 권한을 획득할 수 있다.

<div class="img-left-block">
<img src="/assets/burp_images/Race-Conditions-lab/race-conditions_Lab4-10.png" alt="설명" width="700">
</div>


My Account 페이지에 오면 상단에 **Admin Panel**이 생기는데, 해당 패널에 접속한 후에 `carlos`계정을 삭제하면 lab이 완료된다.


## 정리

이 취약점은 단순히 "요청을 동시에 보냈다"가 아니라,  
**서버가 하나의 값을 여러 요청이 공유하도록 설계되어 있다는 점**에서 발생한다.

email 변경 로직을 보면:

- pending email은 하나만 저장되고  
- 새로운 요청이 들어오면 기존 값을 덮어쓴다  
- 메일 전송은 즉시 처리되지 않고, 이후에 DB 값을 다시 읽어서 생성된다  

이 구조 때문에, 여러 요청이 동시에 들어오면  
각 요청이 서로의 값을 덮어쓰게 되고, 메일 생성 시점에는 전혀 다른 요청의 값이 사용되는 상황이 발생한다.

결과적으로 요청은 각각 따로 보냈지만,  
내부에서는 하나의 shared state를 기준으로 처리되면서 서로 다른 요청의 값이 섞이는 race condition이 발생한다.

이 lab에서 중요한 포인트는 race condition이 반드시 여러 endpoint 사이에서만 발생하는 것이 아니라는 점이다.  
하나의 endpoint라도 내부 처리 과정이 여러 단계로 나뉘어 있고 shared state를 사용하고 있다면 그 자체만으로도 충분히 race window가 생길 수 있다.
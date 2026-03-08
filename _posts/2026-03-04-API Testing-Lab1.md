---
title: API Testing 01 - Exploiting an API endpoint using documentation
description: Documentation을 통한 API 공격을 실습 해보자.
categories: [Web Security, API Testing, Lab]
tags: [웹 보안, 실습]
math: true
mermaid: true
published: true
toc: true
---

# Lab: Exploiting an API endpoint using documentation

## Lab 개요

이 Lab은 취약한 API documentation를 이용하면 어떻게 공격되는지 실행한다.
> 목표: API documentation 을 참조해 `carlos`계정을 삭제

---

## Exploit 과정

사이트에서 사용자의 이메일을 업데이트 할수있는 기능이 있다.

![설명](/assets/burp_images/api-testing-lab/api_lab1-1.png){:width="1200px"}

Burp 에서 요청과 응답이 어떻게 오가는지 확인

![설명](/assets/burp_images/api-testing-lab/api_lab1-2.png){:width="1200px"}
> 요청

![설명](/assets/burp_images/api-testing-lab/api_lab1-3.png){:width="1200px"}
> 응답

사용자의 email input을 PATCH를 통해 업데이트 하는 방식으로 보여짐.

/api/ 디렉토리에 `GET` 요청을 보내면 아래와 같은 Documentation을 반환
![설명](/assets/burp_images/api-testing-lab/api_lab1-4.png){:width="1200px"}

`carlos`계정을 `DELETE` 요청으로 삭제하면서 Lab이 마무리된다.

---

## 취약점 분석

핵심 취약점은 API documentation이 외부에 노출되어 있다는 점이다.  
일반적으로 API documentation은 개발자 편의를 위해 제공되며, API endpoint와 요청 형식, 사용 가능한 HTTP method 등을 상세하게 설명한다.

하지만 이러한 문서가 인증 없이 접근 가능한 상태로 노출될 경우, 공격자는 이를 통해 애플리케이션의 내부 API 구조와 기능을 쉽게 파악할 수 있다.

이번 Lab에서는 `/api/user/wiener` 요청을 분석하는 과정에서 상위 경로인 `/api` 엔드포인트에 접근할 수 있었고, 해당 경로에서 interactive API documentation이 노출되어 있는 것을 확인할 수 있었다.

문서에 포함된 API 목록을 통해 `DELETE /api/user/{username}` 기능을 확인할 수 있었으며, 이를 이용해 `carlos` 계정을 삭제함으로써 Lab을 해결할 수 있었다.

결과적으로 이 취약점은 **공개된 API documentation을 통해 내부 API 기능이 그대로 노출되고, 이를 공격자가 악용할 수 있다는 점**에서 발생한다.

---

## 방어 전략

Production 환경에서는 API documentation을 기본적으로 비활성화하거나, 내부 네트워크 또는 인증된 사용자만 접근할 수 있도록 제한해야 한다.

Swagger나 OpenAPI와 같은 interactive documentation을 사용하는 경우에도 별도의 인증 절차를 요구하거나 관리자 권한을 가진 사용자에게만 접근을 허용하는 것이 바람직하다.

또한 API endpoint 자체에도 적절한 **authorization 검증**이 구현되어야 한다. 모든 요청에 대해 사용자 권한을 확인하여 다른 사용자 계정에 대한 삭제나 수정 요청이 수행되지 않도록 해야 한다.

마지막으로 불필요한 endpoint 노출을 최소화하고, API 경로 탐색 과정에서 내부 구조가 드러나지 않도록 에러 메시지에 민감한 정보를 포함하지 않도록 구성하는 것이 중요하다.
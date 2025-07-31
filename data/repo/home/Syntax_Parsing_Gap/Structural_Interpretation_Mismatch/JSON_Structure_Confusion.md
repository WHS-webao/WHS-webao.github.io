---
title: JSON Structure Confusion
description: 
published: true
date: 2025-07-31T13:23:37.249Z
tags: attack success validation, backend parser profiling, business logic bypass, privilege escalation, sql injection, inconsistent large number decoding, character truncation collision, lenient string parsing, comment truncation, malicious value injection, buffer overflow
editor: markdown
dateCreated: 2025-07-25T10:04:18.749Z
---

# JSON Structure Confusion

## 개요

“JSON Structure Confusion”은 동일한 JSON 데이터 구조를 다양한 파서가 다르게 해석하는 현상을 악용하는 기법이다. 중첩 객체(nested object), flattened key, 배열과 스칼라 값, boolean과 문자열 등 구조와 타입의 해석 방식이 파서마다 상이할 경우, 검증 로직과 실행 로직 사이에 해석 불일치가 발생할 수 있다. 공격자는 이를 이용해 인증 우회, 권한 상승, 로직 회피 등의 조건 분기를 유도하며, 의도한 값이 특정 파서에만 유효하게 처리되도록 구성할 수 있다. 이처럼 구조 해석의 일관성이 보장되지 않을 경우, 파서 간 의미 해석의 단절로 인해 보안 검증이 무력화된다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 구조 해석 불일치 (Structural Interpretation Mismatch)
> → JSON Structure Confusion

## 절차

| Phase          | Description                                                                                                               |
| -------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | JSON 사양(RFC 8259, JSON5, HJSON 등)과 주요 파서(Python, Go, 브라우저 내장 등) 간의 파싱 동작 차이(중복 키, 주석 허용, 트레일링 가비지, 숫자 해석 등)에 대한 정보를 수집한다. |
| Divergence     | 동일한 JSON 페이로드가 서로 다른 파서에서 “다른 값”이나 “다른 오류 처리”를 내도록 유도한다.                                                                  |
| Mutation       | Duplicate-Key 삽입, 숫자 오버플로우/언더플로우, 이스케이프/인코딩 변형, 값 타입 변형 등을 통해 파서별 차이를 크게 만들 수 있도록 페이로드를 구조적으로 변형한다.                       |
| Injection      | 변형한 JSON을 실제 엔드포인트(예: PUT /\_users/... 또는 POST /cart/checkout)에 삽입해서 백엔드 파서까지 도달시킨다.                                      |
| Validation     | 대상 서비스가 요청을 파싱, 검증하는 과정에서 파서 간 해석 차이가 그대로 발생하는지 확인한다.                                                                     |
| Exploitation   | 검증된 파싱 차이를 비즈니스 로직(결제, 인증 등)에 적용해 의도된 공격 행위(과소 청구, 권한 상승 등)를 수행한다.                                                        |
| Exfiltration   | 획득한 권한과 로직 우회를 이용해서 데이터, 금전 이득과 같은 실제 이득을 취한다.                                                                            |

<div style="text-align: left;">
  <img
    src="/json_structure_confusion.png"
    alt="JSON Structure Confusion 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>

## 공격기법

### Reconnaissance

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>API Fingerprinting</td>
      <td>
        <pre><code>{
    "test": 1,
    "test": 2
}</code></pre>
      </td>
      <td>매우 큰 숫자, 소수점, 배열, 중첩 객체 등 다양한 구조와 타입의 JSON 페이로드를 전송한다. 이를 통해 대상 시스템을 구성하는 여러 파서(Python, Go 등)가 숫자나 타입을 어떻게 처리하는지(정수 변환, 부동소수점 오버플로우, 타입 강제 변환 등) 관찰하고, 해석 차이가 발생하는 지점을 식별한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Divergence

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Inconsistent Large Number Decoding</td>
      <td>
        <pre><code>{"qty": 999...999}</code></pre>
      </td>
      <td>매우 큰 정수 값을 사용하여 파서 간의 해석 불일치를 유발한다. 일부 파서는 이 값을 그대로 인식하지만, 다른 파서는 이를 0이나 MAX_INT로 처리하여, 동일한 데이터에 대해 시스템의 두 부분이 서로 다른 값을 인식하게 만든다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Mutation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Character Truncation Collision</td>
      <td>
        <pre><code>{
    "name": "superadmin\ud888"
}</code></pre>
      </td>
      <td>잘못된 유니코드(\ud888)를 키 값에 삽입하여 페이로드를 구조적으로 변형한다. 일부 파서가 이 문자를 파싱 과정에서 제거하는 점을 악용하여, 정상 키(superadmin)와 충돌을 일으키고 권한 상승을 유발한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Lenient String Parsing</td>
      <td>
        <pre><code>{"key": 'value'}</code></pre>
      </td>
      <td>일부 비표준 JSON 파서(예: json-c)가 작은따옴표를 문자열 구분자로 허용하는 점을 악용한다. 이를 통해 표준 파서에서는 거부되는 페이로드를 주입하여, 후속 단계에서 SQL 인젝션과 같은 공격을 유발하는 기반을 마련한다.</td>
      <td>[2]</td>
    </tr>
  </tbody>
</table>

### Injection

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Comment Truncation</td>
      <td>
        <pre><code>obj = {"description": "Duplicate with comments", "test": 2, "extra": /*, "test": 1, "extra2": */}</code></pre>
      </td>
      <td>일부 파서만 인식하는 주석(/* */)을 사용하여 페이로드를 난독화한다. 주석을 지원하지 않는 파서는 이를 일반 문자열로 처리하지만, 지원하는 파서는 주석 안의 중복 키(test: 2)를 인식하여 탐지를 우회하고 숨겨진 값을 전달한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Malicious Value Injection</td>
      <td>
        <pre><code>POST /cart/checkout with {"qty": 999...999}</code></pre>
      </td>
      <td>숫자 처리 방식의 차이를 유발하도록 변형된 JSON 페이로드를 실제 결제 엔드포인트에 주입하여, 백엔드의 검증 로직과 결제 로직이 서로 다른 값을 처리하도록 한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Validation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Divergence Confirmation</td>
      <td>
        <pre><code>GET /api/cart_total</code></pre>
      </td>
      <td>공격 페이로드 전송 후, 시스템의 최종 상태를 확인하는 API 엔드포인트에 요청을 보낸다. 이를 통해 파서 간 불일치로 인해 의도한 결과가 실제로 적용되었는지 검증한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exploitation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Business Logic Bypass</td>
      <td>
        <pre><code>{
    "cart": [
        { "id": 8, "qty": 999999999999999999999999999999999999999999999999999999999999999999999999999999999999999999999999 }
    ]
}</code></pre>
      </td>
      <td>파서 간 큰 숫자 처리 방식의 차이를 악용하여 비즈니스 로직을 우회한다. 앞단의 검증 로직은 큰 수량(qty)을 정상으로 인식하지만, 결제 로직에서는 이를 0으로 처리하여, 실제로는 비용을 지불하지 않고 상품을 주문하는 공격을 수행한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Privilege Escalation</td>
      <td>
        <pre><code>{
    "roles": ["superadmin\ud888"]
}</code></pre>
      </td>
      <td>파서의 문자열 절삭(Truncation) 취약점을 이용해 권한 상승을 시도한다. 권한 검증 시스템은 superadmin과 다른 값으로 인식하여 통과시키지만, 실제 권한을 부여하는 시스템에서는 유니코드(\ud888)가 잘려나가 superadmin으로 인식되어 관리자 권한을 획득한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>SQL Injection via JSON Wrapping</td>
      <td>
        <pre><code>{"credentials": "1', (SELECT sql FROM sqlite_master), '3"}</code></pre>
      </td>
      <td>JSON 파서의 불일치나 관대한 파싱을 통해 주입된 악성 문자열이 백엔드의 SQL 쿼리문에 그대로 삽입되도록 만든다. 이를 통해 데이터베이스 스키마를 유출하거나 데이터를 무단으로 조작하는 SQL 인젝션 공격을 실행한다.</td>
      <td>[2]</td>
    </tr>
    <tr>
      <td>Buffer Overflow via Data Poisoning</td>
      <td>
        <pre><code>{"camera_name": "[ROP_CHAIN]"}</code></pre>
      </td>
      <td>JSON 인젝션을 통해 데이터베이스의 특정 필드에 매우 긴 데이터(예: ROP 체인)를 저장한다. 이후 다른 API 호출이 해당 데이터를 처리하는 과정에서 스택 버퍼 오버플로우를 발생시켜 원격 코드 실행(RCE)을 유발한다.</td>
      <td>[2]</td>
    </tr>
  </tbody>
</table>

### Exfiltration

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Privilege Escalation</td>
      <td>
        <pre><code>HTTP/1.1 200 OK  
Content-Type: text/plain  
Receipt:  
5x Product A @ $100/unit  
1x Product B @ $200/unit  
Total Charged: $300</code></pre>
      </td>
      <td>파서 불일치를 통해 부정하게 얻은 이득(예: 과소 청구된 금액, 무상으로 획득한 상품)이 포함된 최종 영수증이나 거래 완료 결과를 확인함으로써, 공격 성공과 피해 발생을 확정 짓는다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [An Exploration of JSON Interoperability Vulnerabilities](https://bishopfox.com/blog/json-interoperability-vulnerabilities)
* \[2] [https://www.we-fuzz.io/blog/attacking-apis-with-json-injection-a-technical-deep-dive](https://www.we-fuzz.io/blog/attacking-apis-with-json-injection-a-technical-deep-dive)

## 기타 참고문헌

* \[A] [https://github.com/BishopFox/json-interop-vuln-labs/#attack-techniques](https://github.com/BishopFox/json-interop-vuln-labs/#attack-techniques)
* \[B] [https://portswigger.net/web-security/request-smuggling](https://portswigger.net/web-security/request-smuggling)
* \[C] [https://bishopfox.com/blog/json-interoperability-vulnerabilities](https://bishopfox.com/blog/json-interoperability-vulnerabilities)

---
title: JSON Key Duplication
description: 
published: true
date: 2025-07-30T20:24:42.824Z
tags: attack success validation, data exfiltration, backend parser profiling, duplicate key injection, business logic bypass, remote code execution (rce)
editor: markdown
dateCreated: 2025-07-23T10:47:58.341Z
---

# JSON Key Duplication

## 개요

“JSON Key Duplication”은 하나의 JSON 문서에 동일한 키를 중복 정의했을 때, 이를 처리하는 파서 또는 실행 환경이 각기 다르게 해석하는 점을 악용한다. 일반적으로 RFC 8259 등 표준은 중복 키의 동작을 명시하지 않으며, 일부 파서는 첫 번째 값을, 일부는 마지막 값을, 또는 에러로 처리한다. 공격자는 이러한 불일치를 기반으로 검증 파서와 실행 파서 간의 인식 차이를 유도하여, 인증 우회, 권한 상승, 설정 오염, 심지어 원격 코드 실행까지 가능하다. 특히 보안 검증은 중복된 필드 중 무해한 값을 해석하지만, 실제 적용되는 로직은 악의적인 중복 값에 영향을 받도록 구성함으로써, 로직 우회 또는 정책 무력화를 유도하는 것이 핵심이다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 중복 키 처리 불일치 (Duplicate Key Handling Mismatch)
> → JSON Key Duplication

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
    src="/json_key_duplication.png"
    alt="JSON Key Duplication 다이어그램"
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
      <td>중복 키, 주석, 숫자 표현 등 다양한 변형이 포함된 JSON 페이로드를 전송하여, 대상 시스템을 구성하는 여러 파서(Python, Go 등)의 동작 차이를 식별하고 공격 가능한 불일치점을 찾아낸다.</td>
      <td>[2], [3]</td>
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
      <td>Duplicate-Key Precedence</td>
      <td>
        <pre><code>{
    "cart": [
        { "id": 1, 
          "qty": -1, 
          "qty": 1 }
    ]
}</code></pre>
      </td>
      <td>동일한 키(qty)에 상반된 값을 가진 페이로드를 전송하여, 파서별 우선순위(첫 번째 또는 마지막 값 선택) 차이를 유발한다. 이를 통해 앞단의 유효성 검증(Python: qty=1)은 통과하고, 뒷단의 결제 로직(Go: qty=-1)에서는 다른 값으로 해석되도록 만든다.</td>
      <td>[2], [3]</td>
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
      <td>Duplicate Key Insertion</td>
      <td>
        <pre><code>{"is_admin": false, "is_admin": true}</code></pre>
      </td>
      <td>시스템의 논리적 흐름을 조작하기 위해, 애플리케이션이 처리하는 JSON 객체에 의도적으로 동일한 키를 두 번 이상 삽입하여 페이로드를 구조적으로 변형한다.</td>
      <td>[1], [3]</td>
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
      <td>Malicious Configuration Injection</td>
      <td>
        <pre><code>"roles": [],   "roles": ["_admin"]</code></pre>
      </td>
      <td>중복 키로 변형된 JSON 페이로드를 실제 시스템의 엔드포인트(예: NPM 패키지 업로드)에 주입한다. 이를 통해 검증 시스템과 실행 시스템이 서로 다른 설정을 읽도록 유도한다.</td>
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
      <td>[3]</td>
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
        <pre><code>{"charge": 100.00, "charge": 0.00}</code></pre>
      </td>
      <td>결제 시스템에서 파서 간의 불일치를 악용한다. 앞단의 검증 로직(Python)은 마지막 값인 0.00을 보고 결제를 승인하지만, 실제 청구 로직(Go)은 첫 번째 값인 100.00을 사용하여 사용자에게 과도한 요금을 부과하는 등의 금융 로직 공격을 수행한다.</td>
      <td>[3]</td>
    </tr>
    <tr>
      <td>Remote Code Execution (RCE)</td>
      <td>
        <pre><code>{
    "name": "chloe",
    "password": "password",
    "roles": [],
    "roles": ["_admin"],
    "type": "user"
}</code></pre>
      </td>
      <td>NPM 레지스트리에서 CouchDB(첫 번째 값 유지)와 JavaScript(마지막 값 유지)의 파싱 차이를 악용한다. 검증 단계에서는 비어있는 roles를 보고 통과시키지만, 실제 설치 시에는 악성 코드가 포함된 두 번째 roles가 실행되어 원격 코드 실행을 유발한다.</td>
      <td>[1]</td>
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
      <td>Data Exfiltration via RCE</td>
      <td>
        <pre><code>cat /etc/password</code></pre>
      </td>
      <td>파서 불일치를 통해 부정하게 얻은 이득(예: 과소 청구된 금액, 무상으로 획득한 상품)이 포함된 최종 영수증이나 거래 완료 결과를 확인함으로써, 공격 성공과 피해 발생을 확정 짓는다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [https://justi.cz/security/2017/11/14/couchdb-rce-npm.html](https://justi.cz/security/2017/11/14/couchdb-rce-npm.html)
* \[2] [https://0day.click/parser-diff-talk-oc25/](https://0day.click/parser-diff-talk-oc25/)
* \[3] [https://bishopfox.com/blog/json-interoperability-vulnerabilities](https://bishopfox.com/blog/json-interoperability-vulnerabilities)

## 기타 참고문헌

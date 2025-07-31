---
title: IDN Homograph Spoofing
description: 
published: true
date: 2025-07-31T14:43:37.800Z
tags: target & language analysis, homograph domain crafting, client-side rendering deception, browser/client behavior check, certified phishing site deployment, credential harvesting
editor: markdown
dateCreated: 2025-07-25T10:18:07.453Z
---

# IDN Homograph Spoofing

## 개요

“IDN Homograph Spoofing”은 국제화 도메인 이름(IDN)에서 동형 문자와 퓨니코드(puny-code)의 변환 순서 차이를 악용해, 브라우저・메일・오피스 클라이언트 화면에 실제 서비스 주소와 거의 구별되지 않는 가짜 도메인을 표시하도록 만드는 기법이다. 각 소프트웨어가 IDN->유니코드 정규화와 폰트 렌더링을 다르게 처리해 동일 문자열 검증이 무력화 되고, 공격자는 이 위장 주소를 피싱 메일・문서 등에 넣어 사용자를 가짜 사이트로 유인한다. 최종적으로 로그인 자격 증명, 세션 토큰, 결제 정보가 탈취돼 계정 장악과 금전적 손실로 이어진다.

## 분류

> 데이터 표현 모델 불일치 (Data Repretation Model Gap)
> → 유사문자 및 정규화 처리 문제 (Homoglyph & Normalization Mismatch)
> → IDN Homograph Spoofing

## 절차

| Phase          | Description                                                                               |
| -------------- | ----------------------------------------------------------------------------------------- |
| Reconnaissance | 타겟 브랜드, 도메인을 조사하고 대상 사용자의 사용 언어 및 브라우저를 파악한다.                                             |
| Mutation       | 시각적으로 비슷한 유니코드 문자를 사용해서 도메인을 변형한다. 예시로는  аpple.com 과 같이 일반 사용자는 거의 구분할 수 없도록 변형한다.        |
| Evasion        | 변형된 도메인은 실제로 등록 후에 Punycode로 변환된다.  이를 숨기기 위해 리다이렉션, 이메일 링크 등을 사용해서 보안 장비나 사용자의 주의를 회피한다. |
| Validation     | 브라우저에서 실제로 도메인이 유사하게 표시되는지 확인하고 보안 경고/IDN 차단 우회 여부를 확인한다.                                 |
| Exploitation   | 피싱 페이지(로그인 폼, MFA 입력창 등)를 생성하고 악성 스크립트를 삽입한다.                                             |
| Exfiltration   | 수집된 계정 정보를 전송하거나 세션 탈취, 인증 우회를 수행한다.                                                      |

<div style="text-align: left;">
  <img
    src="/idn_homograph_spoofing.png"
    alt="IDN Homograph Spoofing 다이어그램"
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
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Target &amp; Language Analysis</td>
      <td>
        <pre><code>example.com &amp; Cyrillic alphabet</code></pre>
      </td>
      <td>공격 대상 기업과 해당 기업의 사용자가 주로 사용하는 언어를 식별한다. 이후, 라틴 문자와 시각적으로 유사한 문자(Homoglyph)가 포함된 다른 스크립트(예: 키릴 문자, 그리스 문자)를 분석하여 위조에 사용할 문자를 찾아낸다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Mutation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Homograph Domain Crafting</td>
      <td>
        <pre><code>аpple.com (Cyrillic 'а')
bițdefender.com (Romanian 'ț')</code></pre>
      </td>
      <td>키릴 문자의 'а'를 라틴 문자의 'a'로 대체하거나, 특정 언어의 유사 문자(예: 루마니아어의 ț를 t 대신 사용)를 사용하여, 육안으로는 원본과 거의 구별할 수 없는 위조 도메인을 생성한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Evasion

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Client-Side Rendering Deception</td>
      <td>
        <pre><code>xn--pple-43d.com → аpple.com</code></pre>
      </td>
      <td>위조된 도메인을 퓨니코드(Punycode)로 등록한 뒤, 이메일 클라이언트(Outlook)나 특정 브라우저(Firefox)가 퓨니코드를 그대로 표시하지 않고, 사용자를 속이는 유니코드 동형 문자로 렌더링하는 점을 악용한다. 사용자는 링크 위에 마우스를 올렸을 때 정상적인 주소로 착각하게 된다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Validation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Browser/Client Behavior Check</td>
      <td>
        <pre><code>Firefox vs. Microsoft Edge</code></pre>
      </td>
      <td>공격자는 생성한 위조 도메인 링크를 다양한 클라이언트에서 열어보며, 어떤 환경에서 퓨니코드가 아닌 위조된 유니코드 도메인이 표시되는지 확인한다. 이를 통해 보안 경고를 우회하고 피싱 공격이 성공할 수 있는 취약한 대상을 선별한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exploitation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Certified Phishing Site Deployment</td>
      <td>
        <pre><code>https://xn--n1aag8f.com (оорѕ.com) with a valid certificate</code></pre>
      </td>
      <td>위조된 퓨니코드 도메인에 대해 유효한 SSL/TLS 인증서를 발급받아 피싱 사이트에 적용한다. 사용자는 브라우저에 표시된 위조된 주소와 자물쇠 아이콘을 보고 사이트를 신뢰하게 되어, 로그인 정보나 금융 정보를 입력하도록 유도된다.</td>
      <td>[1], [A]</td>
    </tr>
  </tbody>
</table>

### Exfiltration

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Credential Harvesting</td>
      <td>
        <pre><code>POST /login on the phishing site</code></pre>
      </td>
      <td>피싱 사이트의 로그인 폼을 통해 사용자가 입력한 계정 정보, 비밀번호, MFA 코드 등을 탈취한다. 탈취된 정보는 공격자가 제어하는 서버로 전송되어 계정 장악이나 금전적 사기에 사용된다.</td>
      <td>[1], [A]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [https://www.bitdefender.com/en-us/blog/businessinsights/homograph-phishing-attacks-when-user-awareness-is-not-enough](https://www.bitdefender.com/en-us/blog/businessinsights/homograph-phishing-attacks-when-user-awareness-is-not-enough)

## 기타 참고문헌

* \[A] [https://www.bitdefender.com/en-us/blog/labs/new-homograph-phishing-attack-impersonates-bank-of-valletta-leverages-valid-tls-certificate](https://www.bitdefender.com/en-us/blog/labs/new-homograph-phishing-attack-impersonates-bank-of-valletta-leverages-valid-tls-certificate)

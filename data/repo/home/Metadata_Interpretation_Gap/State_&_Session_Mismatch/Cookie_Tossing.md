---
title: Cookie Tossing
description: 
published: true
date: 2025-07-30T18:31:49.876Z
tags: privileged data access, subdomain control & target identification, session cookie tossing, session context poisoning, csrf & samesite policy bypass, session contamination verification, oauth flow hijacking
editor: markdown
dateCreated: 2025-07-25T10:19:18.848Z
---

# Cookie Tossing

## 개요

“Cookie Tossing”은 클라이언트와 서버가 쿠키의 도메인 범위(domain scope)를 해석하는 방식에 차이가 존재할 때 발생하는 취약점이다. 공격자는 제어 가능한 도메인에서 쿠키를 설정하여, 상위 도메인에 대한 쿠키로 전달되도록 유도하고, 서버가 이를 정상적인 인증 정보로 오인하도록 만든다. 이 과정은 쿠키의 도메인 속성 처리에 대한 클라이언트와 서버 간 해석 불일치에서 비롯되며, 인증 흐름이나 세션 관리 로직이 이 차이를 고려하지 않을 경우, 인증 우회나 권한 탈취로 이어질 수 있다.

## 분류

> 메타데이터 해석 불일치 (Metadata Interpretation Gap)
> → 상태·세션 해석 불일치 (State & Session Mismatch)
> → Cookie Tossing

## 절차

| Phase          | Description                                                                                         |
| -------------- | --------------------------------------------------------------------------------------------------- |
| Reconnaissance | 공격자는 XSS,서브도메인 takeover 등으로 제어 가능한 하위 도메인을 확보하고, 대상 서비스의 민감한 OAuth / API 엔드포인트를 식별한다.               |
| Injection      | 확보한 하위 도메인에서 Domain=.example.com 및 Path=/민감경로가 지정된 공격자 세션 쿠키를 피해자 브라우저에게 “토스”한다.                    |
| Poisoning      | 피해자 브라우저는 동일 사이트로 인식하여 해당 경로에 접근할 때 우선순위가 더 높은 공격자 쿠키를 자동으로 포함한다. 그 결과, 피해자가 보내는 요청이 공격자 세션으로 처리된다. |
| Bypass         | 대부분의 JSON API는 CSRF 토큰 없이 SOP + CORS에만 의존하며 SameSite 쿠키도 서브도메인 간 공격을 차단하지 못하므로 쿠키 토스가 그대로 통한다.      |
| Validation     | 공격자는 가벼운 조회 요청을 보내서 세션이 실제로 오염되었는지 확인한다.                                                            |
| Exploitation   | OAuth 인증, 결제, 권한 변경 등 상태 변경 요청이 공격자 세션으로 실행되어 계정 가로채기·권한 탈취가 발생한다.                                  |
| Exfiltration   | 공격자는 획득한 세션으로 민감 데이터 열람·수정 및 외부 서비스 연동 등 광범위한 접근권을 획득한다.                                            |

<div style="text-align: left;">
  <img
    src="/cookie_tossing.png"
    alt="Cookie Tossing 다이어그램"
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
      <td>Subdomain Control &amp; Target Identification</td>
      <td>
        <pre><code>sub.example.com
/api/authorize</code></pre>
      </td>
      <td>서비스의 기본 기능이나 XSS 취약점을 이용해 제어 가능한 하위 도메인을 확보한다. 동시에, 공격에 사용할 OAuth 인증, API 엔드포인트 등 민감한 경로를 식별한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Injection

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
      <td>Session Cookie Tossing</td>
      <td>
        <pre><code>document.cookie = "session-id=ATTACKER_ID; domain=.example.com"</code></pre>
      </td>
      <td>공격자가 제어하는 하위 도메인에서, 자신의 세션 쿠키(session-id)를 상위 도메인(domain=.example.com)과 특정 민감 경로(path=/api/authorize)를 대상으로 하여 피해자의 브라우저에 주입(Toss)한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Poisoning

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
      <td>Session Context Poisoning</td>
      <td>
        <pre><code>Cookie: session-id=ATTACKER_ID</code></pre>
      </td>
      <td>피해자가 민감한 경로에 접근하면, 브라우저는 우선순위가 더 높은 공격자의 쿠키를 요청에 자동으로 포함시킨다. 그 결과, 서버는 해당 요청을 피해자가 아닌 공격자의 세션 컨텍스트에서 처리하게 되어 세션이 오염된다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Bypass

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
      <td>CSRF &amp; SameSite Policy Bypass</td>
      <td>
        <pre><code>SameSite=Lax</code></pre>
      </td>
      <td>대부분의 JSON 기반 API가 SOP에만 의존하고 CSRF 토큰을 사용하지 않는 점을 악용한다. 또한, 하위 도메인에서의 요청은 동일 사이트(Same-Site)로 간주되므로, SameSite=Lax나 Strict 같은 쿠키 속성도 이 공격을 방어하지 못한다.</td>
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
      <td>Session Contamination Verification</td>
      <td>피해자의 외부 계정이 공격자의 계정에 나타난다.</td>
      <td>공격자는 자신의 계정에서 피해자의 리소스(예: 연동된 외부 서비스 계정)가 나타나는지 확인함으로써 세션이 성공적으로 오염되었는지 검증한다.</td>
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
      <td>OAuth Flow Hijacking</td>
      <td>피해자의 외부 계정 → 공격자의 계정</td>
      <td>오염된 세션을 통해 피해자가 OAuth 인증을 시도하면, 피해자의 외부 서비스 계정(GitHub, BitBucket 등)이 공격자의 계정에 연동되어 접근 권한을 탈취한다.</td>
      <td>[1]</td>
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
      <td>Privileged Data Access &amp; Modification</td>
      <td>-</td>
      <td>성공적으로 탈취한 OAuth 권한을 이용해, 피해자의 비공개 소스 코드 저장소를 읽고 수정하거나, 계정 내의 민감한 정보에 접근하여 유출한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [https://labs.snyk.io/resources/hijacking-oauth-flows-via-cookie-tossing/](https://labs.snyk.io/resources/hijacking-oauth-flows-via-cookie-tossing/)

## 기타 참고문헌

* \[A] [https://www.youtube.com/watch?v=xLPYWim60jA](https://www.youtube.com/watch?v=xLPYWim60jA)

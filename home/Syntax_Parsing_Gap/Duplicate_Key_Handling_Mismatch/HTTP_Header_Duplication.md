---
title: HTTP_Header_Duplication
description: 
published: true
date: 2025-07-31T21:52:15.215Z
tags: 
editor: markdown
dateCreated: 2025-07-20T11:31:41.164Z
---

# HTTP Header Duplication

## 개요

“HTTP Header Duplication”는 HTTP 요청 또는 응답에서 특정 헤더 필드가 여러 번 나타날 때, 각 요소가 이를 어떻게 파싱하고 해석하는지에 대한 일관성 문제에서 발생하는 취약점이다. RFC 7230에서는 일부 헤더(예: Cache-Control)의 경우, 필드값을 쉼표로 결합해 처리할 수 있게 허용하지만, 실제 환경에서는 프록시・브라우저・WAF・애플리케이션・라이브러리 등 각 요소가 헤더 중복을 무시, 결합, 순서 변경, 또는 마지막값만 적용하는 등 다르게 처리하는 경향이 있다. 이러한 파싱 불일치는 인증/인가 정책, 보안 필터링, 캐시 로직 등의 기능을 우회하거나 오작동하게 만들 수 있으며, 공격자는 이를 악용한 권한 우회, XSS, 캐싱 문제, 부적절한 요청 처리가 가능하다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 중복 키 처리 불일치 (Duplicate Key Handling Mismatch)
> → HTTP Header Duplication

## 절차

| Phase          | Description                                                                                                            |
| -------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 클라이언트 → Ingress → 백엔드로 이어지는 다단계 파서 구조를 파악하고, 각 계층이 중복 헤더를 어떤 규칙으로 처리하는지 탐색한다.                                          |
| Mutation       | 대소문자, 공백, 0x00 등으로 헤더 이름/값을 변형하여 탐지를 우회한다.                                                                             |
| Divergence     | 한 요청에 동일 헤더 이름을 두 번 넣어서 파서 간 해석 차이를 유발한다. 예: Authorization: Bearer fake\_admin 다음 줄에 Authorization: Bearer legit\_user |
| Validation     | 각 계층의 로그, 응답을 관찰해서 어느 헤더를 선택했는지 확인한다. 예를 들어 프론트 파서는 무해한 토큰을, 백엔드는 공격 토큰을 사용하는 것을 확인한다.                                 |
| Exploitation   | 검증된 페이로드로 실제 업무 API를 호출하고, 백엔드가 높은 권한으로 요청을 처리해서 인증 우회, 권한 상승이 발생한다.                                                   |
| Exfiltration   | 민감 데이터(세션 토큰, 서비스 계정 자격 등)를 수집한다.                                                                                      |

<div style="text-align: left;">
  <img
    src="/http_header_duplication.png"
    alt="HTTP Header Duplication 다이어그램"
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
      <td>Duplicate Header Precedence Probing</td>
      <td>
        <pre><code>GET /example HTTP/1.1
Host: vulnerable-website.com
Host: bad-stuff-here</code></pre>
      </td>
      <td>한 요청에 동일 헤더, 다른 값을 부여하고, 각 계층이 중복 헤더를 무시하는지, 결합하는지, 첫번째/마지막을 선택하는지 파악한다.</td>
      <td>[A]</td>
    </tr>
    <tr>
      <td>Duplicate Header Tolerance Check</td>
      <td>-</td>
      <td>탐지 우회 payload가 아니더라도 무해한 헤더를 두 번 포함한 요청을 전송하고, WAF나 서버가 에러를 반환하는지 확인한다.</td>
      <td>[A]</td>
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
      <td>Case Variation in Header Name</td>
      <td>
        <pre><code>partner_key
partner_Key
Partner_Key
PARTNER_KEY</code></pre>
      </td>
      <td>HTTP 표준상 헤더 이름은 대소문자를 구분하지 않지만, 전처리 단계에서 대소문자가 다르면 별개 헤더로 인식되어, 중복 탐지를 피할 수 있다.</td>
      <td>[B]</td>
    </tr>
    <tr>
      <td>Header Folding / Indentation</td>
      <td>
        <pre><code>GET /example HTTP/1.1
    Host: bad-stuff-here
Host: vulnerable-website.com</code></pre>
      </td>
      <td>중복 헤더 중 하나를 줄바꿈 뒤 공백이나 탭으로 시작되도록 보내 WAF의 파싱을 교란한다.</td>
      <td>[A]</td>
    </tr>
  </tbody>
</table>

### Divergence

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
      <td>Duplicate Authorization Token</td>
      <td>
        <pre><code>GET /api/resource HTTP/1.1
Host: api.example.com
Authorization: Bearer $alg_none_fake_admin_JWT
Authorization: Bearer $legit_JWT
Content-Type: application/json
Accept: application/json</code></pre>
      </td>
      <td>하나의 요청에 두 개의 Authorization 헤더를 넣어, 프록시/인증 게이트웨이와 백엔드 애플리케이션 간 토큰 해석 불일치를 일으킨다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Poisoning with Duplication</td>
      <td>
        <pre><code>{
  "tenants": [
    "tenantB",
    "tenantA"
  ],
  "tokens": [
    "tenantA"
  ],
  "authorizerContext": {
    "authPolicyCreatedForTenant": "tenantA",
    "authPolicyCreatedForToken": "tenantA",
    "authPolicyCreatedAt": "2023-01-24T12:25:42.289Z"
  }
}</code></pre>
      </td>
      <td>인증에 사용되는 헤더를 두 번 보내 캐시가 잘못된 키로 정책을 적용하게 만든다. 예시의 경우, 게이트웨이는 이전에 캐시된 권한 정책(tenant A용)을 재사용하여 tenant B에 대한 요청임에도 불구하고 검증을 통과시킨다.</td>
      <td>[2]</td>
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
      <td>Differential Logging Analysis</td>
      <td>
        <pre><code>Authorizer 로그 : X-Tenant: A(첫 번째 헤더), X-Amzn-Remapped-X-Tenant: B(두 번째 헤더)
최종 서비스 로그 : X-Tenant: B
(예: AWS API Gateway의 Lambda Authorizer)</code></pre>
      </td>
      <td>각 단계의 로그를 통해 헤더 우선순위를 파악하고 공격이 성공적으로 조건을 충족했는지 입증한다.</td>
      <td>[2]</td>
    </tr>
    <tr>
      <td>Response Discrepancy Observation</td>
      <td>-</td>
      <td>동일한 요청을 헤더 순서만 바꿔 두 번 보내고 응답 차이를 관찰한다. 응답상의 차이를 통해 파서 간 해석 차이가 실제로 발생했는지 검증하고, 나아가 발생 조건도 파악 가능하다.</td>
      <td>[2]</td>
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
      <td>Authentication Bypass</td>
      <td>
        <pre><code>GET /api/resource HTTP/1.1
Host: api.example.com
Authorization: Bearer $alg_none_fake_admin_JWT
Authorization: Bearer $legit_JWT
Content-Type: application/json
Accept: application/json</code></pre>
      </td>
      <td>검증된 중복 헤더(예: Authorization) 페이로드를 실제 관리자 기능 호출에 사용한다. 프론트 보안장치는 요청을 통과시키고, 백엔드는 관리자 토큰에 따라 관리자 권한으로 요청을 처리한다.</td>
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
      <td>Target Data Dump</td>
      <td>-</td>
      <td>AWS API Gateway 사례에서는 Tenant 간 경계가 흐트러진 상황에서 공격자가 다른 Tenant의 모든 데이터를 API로 추출 가능했다.</td>
      <td>[2]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [https://0day.click/parser-diff-talk-oc25/](https://0day.click/parser-diff-talk-oc25/)
\[2] [https://securityblog.omegapoint.se/en/writeup-apigw/](https://securityblog.omegapoint.se/en/writeup-apigw/)

## 기타 참고문헌

\[A] [https://portswigger.net/web-security/host-header/exploiting#how-to-test-for-vulnerabilities-using-the-http-host-header](https://portswigger.net/web-security/host-header/exploiting#how-to-test-for-vulnerabilities-using-the-http-host-header)
\[B] [https://discuss.google.dev/t/case-sensitive-issue-while-setting-and-getting-request-headers-using-js-policy/139707](https://discuss.google.dev/t/case-sensitive-issue-while-setting-and-getting-request-headers-using-js-policy/139707)

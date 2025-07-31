---
title: Clipjacking
description: 
published: true
date: 2025-07-31T15:43:29.352Z
tags: arbitrary account takeover, vulnerability identification, social engineering camouflage, attack success validation, data exfiltration, cookie inspection, framing bypass, persistent focus hijack, cursorjacking, invisible layering, clipboard data capture, malicious data injection, response time analysis
editor: markdown
dateCreated: 2025-07-22T16:50:20.751Z
---

# Clipjacking
## 개요
“Clipjacking”은 사용자가 웹 페이지 상에서 복사(Copy)한 콘텐츠와 실제 클립보드에 저장되는 데이터가 불일치하도록 만들어, 시각적 UI와 입력 데이터 간의 컨텍스트 경계를 교란하는 공격 기법이다. 공격자는 `execCommand('copy')`, Clipboard API, 투명 텍스트 오버레이 등의 기술을 이용해 사용자가 의도한 텍스트 대신 공격자가 정의한 값을 클립보드에 삽입하도록 유도할 수 있다. 사용자는 화면에 표시된 내용을 복사했다고 믿지만, 실제 붙여넣기 시 공격자가 설정한 악성 URL, 주소, 명령어 등이 입력되어 피싱, 명령 실행, 세션 탈취 등으로 이어질 수 있다.


## 분류

> 컨텍스트 경계 불일치 (Context Boundary Gap)  
> → 포커스 및 입력 컨텍스트 불일치 (Input/Focus Mismatch)  
> → Clipjacking

## 절차

| Phase               | Description                                                                                                                                                                               |
|---------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Reconnaissance      | 대상 사이트가 `X-Frame-Options` 또는 `CSP frame-ancestors`가 미설정인지 확인해 iframe 로드 가능 여부를 점검한다. `SameSite=None` 쿠키의 존재를 조사하고, `Domain/Path/HttpOnly` 범위를 파악한다. |
| Bypass              | 프레이밍이 막혀있으면 `sandbox` iframe에 `allow-same-origin`을 붙이거나 `window.name` 트릭을 사용해 우회한다. 이어서 세션 쿠키를 탈취해 HTTPS로 재주입하거나, 피싱으로 피해자를 로그인 상태로 만들어둔다.       |
| Injection           | 공격 페이지에 iframe을 숨겨서 iframe으로`https://target.com`을 로드한다. `setInterval(()=>iframe.contentWindow.focus(), 250)` 등으로 포커스를 유지하도록 한다.                                        |
| Obfuscation         | 상위 페이지에 “Copy to continue” 같은 가짜 버튼이나 간단한 게임 요소를 덮어씌워 사용자의 클릭을 유도한다. 사용자가 버튼을 누른 순간에만 `navigator.userActivation.isActive`를 확인해 동작을 실행함으로써, 브라우저가 요구하는 사용자 제스처 조건을 만족시킨다.         |
| Capture             | 사용자가 가짜 버튼을 클릭했을 때 상위 문서 핸들러에서 `await navigator.clipboard.readText()`로 클립보드 내용을 수집한다. 동시에 iframe 안의 입력 칸을 선택하도록 유도하고 사용자가 `Ctrl+A`, `C`를 직접 누르게 해 민감 데이터가 클립보드에 올라오게 만든다. |
| Clipboard Injection | 같은 이벤트 안에서 `navigator.clipboard.writeText(maliciousPayload)`를 실행해 악성 주소나 명령을 클립보드에 덮어쓴다. (쓰기 역시 user activation 또는 Clipboard-write 권한 필요) 이후 `Ctrl+V`를 누르도록 유도하여 피해자가 다른 곳에 악성 데이터를 붙여놓도록 한다.   |
| Validation          | `readText()`를 한 번 더 호출해 가져온 문자열을 정규식으로 검사하여 길이와 패턴이 기대한 값인지 확인한다. 필요하면 iframe에서 `postMessage`를 보내 상태코드나 리다이렉트 URL 같은 실행 결과를 상위 페이지에서 받아 검증한다.|
| Exfiltration        | 탈취한 토큰, OTP, 지갑 주소 등을 `sendBeacon()`이나 WebSocket으로 데이터를 빼낸다.                                                                                                          |

<div style="text-align: left;">
  <img
    src="/clipjacking.png"
    alt="Clipjacking 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>

## 공격 기법

### Reconnaissance

| 공격 기법                    | 예시                          | 설명                                                                                                                                             | 출처    |
|------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|--------|
| Vulnerability Identification | `X-Frame-Options: (not set)`  | 프레이밍 방지 헤더(`X-Frame-Options`)가 없는 것을 확인하여 iframe 로드가 가능한지 탐지한다.                                                                           | [1], [3] |
| Cookie Inspection            | `SameSite=None; Secure`       | `SameSite=None` 속성을 가진 세션 쿠키를 포함한 HTTP 응답 헤더를 분석하여, 크로스 사이트 컨텍스트(즉, `iframe` 내부)에서도 인증 정보가 전송되는지 확인한다. | [1]    |

### Bypass

| 공격 기법     | 예시                                 | 설명                                                                                                  | 출처  |
|---------------|--------------------------------------|-------------------------------------------------------------------------------------------------------|------|
| Framing Bypass | `<iframe sandbox="allow-scripts">`  | `sandbox` 속성을 이용하여 프레이밍 방지 헤더를 우회하고, iframe 내에 대상 사이트를 렌더링한다.           | [1]  |

### Injection

| 공격 기법               | 예시                              | 설명                                                                                                                               | 출처  |
|-------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------------|------|
| Persistent Focus Hijack | `setInterval(() => iframe.focus())` | `setInterval` 함수를 사용하여 숨겨진 iframe에 지속적으로 포커스를 강제하고, 사용자의 모든 키보드 입력 이벤트를 보이지 않는 `iframe`으로 전달한다.             | [1]  |
| Cursorjacking           | `cursor: none;`                   | CSS/JS를 이용해 커서 모양이나 위치를 조작하여, 사용자가 실제 클릭하는 지점과 시각적으로 보이는 커서 위치를 다르게 만든다.           | [1]  |
| Account Takeover        | `email=attacker@example.com`      | 이메일 변경 페이지 등 iframe 프레이밍이 가능한 기능을 악용하여, 사용자는 다른 클릭을 유도당하면서 실제로는 비밀번호 재설정이나 이메일 변경 버튼을 클릭하게 하여 계정 탈취를 시도한다. | [1]  |

### Obfuscation

| 공격 기법                   | 예시               | 설명                                                                                                                                                  | 출처  |
|-----------------------------|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| Social Engineering Camouflage | Fake CAPTCHA UI  | **UI Layer:** 가짜 CAPTCHA나 이벤트 UI를 노출시켜 사용자가 상호작용하도록 유도한다.<br>**Clipboard Layer:** 사용자의 클릭과 같은 상호작용을 트리거로 사용하여, 보이지 않는 `iframe`이 클립보드 제어 권한을 획득하도록 한다.    | [2]  |
| Invisible Layering          | `opacity: 0; z-index: 100;` | CSS 속성으로 공격 대상 `iframe`을 사용자에게 보이지 않게(투명하게) 만들고, 그 위에 가짜 UI 레이어를 덮어씌워 사용자의 클릭을 유도한다.                                           | [3]  |

### Capture

| 공격 기법               | 예시                              | 설명                                                                                                                | 출처  |
|-------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------------|------|
| Clipboard Data Capture  | `navigator.clipboard.readText()` | 포커스가 탈취된 상태에서 `readText()` API를 호출하여 브라우저의 허용 팝업 없이 클립보드 내용을 몰래 읽어온다. 이를 통해 세션 토큰이나 개인정보를 탈취하여 세션 하이재킹으로 이어진다. | [1]  |

### Clipboard Injection

| 공격 기법                | 예시                         | 설명                                                                                                                                       | 출처    |
|--------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|--------|
| Malicious Data Injection | `clipboard.writeText(ATTACKER_ADDR)` | UI Layer: 화면에 보이는 정상 텍스트를 복사한다고 믿게 한다.<br>Clipboard Layer: 실제로 `writeText()`가 동작해 클립보드를 악성 페이로드로 교체한다. | [2], [3] |

### Validation

| 공격 기법                  | 예시                                    | 설명                                                                                                                   | 출처  |
|----------------------------|-----------------------------------------|------------------------------------------------------------------------------------------------------------------------|------|
| Attack Success Validation  | `if (data.startsWith('session_token='))` | 클립보드에서 읽어온 데이터가 세션 토큰과 같은 의도한 형식과 일치하는지 확인하여 공격 성공 여부를 최종 검증한다.                                    | [1]  |
| Response Time Analysis     | `performance.now()`                    | 사용자가 숨겨진 `iframe` 내에서 상호작용한 후의 응답 시간을 측정하여, 실제로 클릭이나 데이터 복사 같은 행위가 있었는지 추정함으로써 공격 성공 여부를 간접적으로 검증한다.       | [1]  |

### Exfiltration

| 공격 기법           | 예시                               | 설명                                                                                      | 출처  |
|---------------------|------------------------------------|-----------------------------------------------------------------------------------------|------|
| Data Exfiltration   | `navigator.sendBeacon(...)`        | 탈취한 민감한 데이터를 `sendBeacon()` API 등을 이용해 외부 공격자 서버로 비동기적으로 전송하여 최종 유출을 완료한다. | [1]  |

---

## 분류에 해당하는 문서
- [1] [Clipjacking: Hacked by copying text - Clickjacking but better](https://blog.jaisal.dev/articles/cwazy-clipboardz)  
- [2] [Malwarebytes Blog: Fake CAPTCHA websites hijack your clipboard to install information stealers](https://www.malwarebytes.com/blog/news/2025/03/fake-captcha-websites-hijack-your-clipboard-to-install-information-stealers)  
- [3] [Bufferzone Security: Clipboard Hijacking Attacks and How to Prevent Them](https://bufferzonesecurity.com/clipboard-hijacking-attacks-and-how-to-prevent-them/)  

## 기타 참고 문헌
- [A] [Trend Micro: Unmasking Fake CAPTCHA Cases](https://www.trendmicro.com/en_us/research/25/e/unmasking-fake-captcha-cases.html)  
- [B] [MITRE ATT&CK T1414: Clipboard Data Techniques](https://attack.mitre.org/techniques/T1414/)  
- [C] [The Hacker News: Binance Warns of Rising Clipper Malware Attacks Targeting Cryptocurrency Users](https://thehackernews.com/2024/09/binance-warns-of-rising-clipper-malware.html)

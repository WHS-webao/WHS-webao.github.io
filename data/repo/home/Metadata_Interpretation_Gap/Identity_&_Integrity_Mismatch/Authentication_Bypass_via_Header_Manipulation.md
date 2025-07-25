---
title: Authentication Bypass via Header Manipulation
description: 
published: true
date: 2025-07-25T15:51:07.615Z
tags: 
editor: markdown
dateCreated: 2025-07-23T11:14:19.304Z
---

# Authentication Bypass via Header Manipulation

## 개요
“Authentication Bypass via Header Manipulation”은 Microsoft Exchange Server에서 `SecurityToken`이라는 인증 관련 헤더를 프록시 구성 요소와 백엔드 구성 요소가 서로 다르게 해석하는 메타데이터 해석 불일치다. 프론트엔드는 해당 헤더를 인증 확인 없이 통과시키고, 백엔드는 이 메타데이터를 신뢰된 인증 정보로 간주하여 요청을 처리한다. 이로 인해 공격자는 인증 없이 임의 사용자의 메일 설정 페이지에 접근하고 포워딩 설정 등을 변경할 수 있으며, 결과적으로 인증 우회 및 민감 정보 유출이 발생할 수 있다.

## 분류

> 메타 데이터 해석 불일치 (Metadata Interpretation Gap)  
> → 신원·무결성 해석 불일치 (Identity & Integrity Mismatch)  
> → Authentication Bypass via Header Manipulation

## 절차

| Phase            | Description                                                                                                                                                                |
|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Reconnaissance   | 443 포트 스캔, `Server:` 헤더, AutoDiscover 등으로 Exchange를 확인하여 `/ecp` 노출 여부와 버전을 식별한다.                                                                  |
| Divergence       | `SecurityToken=ANY` 쿠키를 삽입하여 프론트엔드와 백엔드 해석 차이를 유발하여 인증 공백을 확보한다.                                                                            |
| Injection        | 존재하지 않는 JS 파일을 요청해 HTTP 500 오류를 받고 응답 본문에서 유효한 canary 값을 획득한다. canary를 포함한 POST로 `/ecp` 액션을 실행한다.                                 |
| Validation       | 변조된 토큰을 포함한 요청에 대해 서버가 200 OK 응답을 반환하는지 확인하여, 명령 실행이 성공했음을 검증한다.                                                                   |
| Persistence      | 장기적 접근을 위해 메일함에 ‘숨김(또는 영구) 전달 규칙, 대리 권한, 백도어 계정’을 심어두어 서버 재부팅, 패치 이후에도 접근을 유지한다.                                            |
| Exploitation     | 모든 수신 메일을 외부 주소로 자동으로 전달하는 등 메일 전달 규칙과 권한을 변경한다.                                                                                         |
| Exfiltration     | 공격자 M365 계정, SMTP 서버에서 자료를 수집하여 메일을 실시간으로 수집한다.                                                                                                |
| Celanup          | 로그 흔적을 은폐하거나 노이즈 주입(Confusion)으로 포렌식을 방해한다.                                                                                                       |

<div style="text-align: left;">
  <img
    src="/authentication_bypass_via_header_manipulation.png"
    alt="Authentication Bypass via Header Manipulation 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>

## 공격 기법

### Reconnaissance

| 공격 기법               | 예시                          | 설명                                                                                                                      | 출처 |
|-------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------|------|
| Exchange Version Check  | `GET /EWS/Exchange.asmx`      | `/EWS/Exchange.asmx` 경로에 요청을 보내고, HTTP 응답 헤더에서 `X-OWA-Version` 값을 확인하여 Exchange 서버의 버전을 식별한다. | [B]  |

### Devergence

| 공격 기법                      | 예시                         | 설명                                                                                                                                                      | 출처 |
|--------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| SecurityToken Cookie Injection | `Cookie: SecurityToken=x;`   | HTTP 요청에 임의의 값을 가진 `SecurityToken` 쿠키를 포함시킨다. 프론트엔드는 이를 보고 인증 처리를 백엔드로 위임하지만, 백엔드는 관련 모듈 부재로 인증을 수행하지 않아 해석 불일치가 발생한다. | [1]  |

### Injection

| 공격 기법             | 예시                                 | 설명                                                                                                                           | 출처 |
|-----------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|------|
| Canary Token Leakage  | `GET /ecp/nonexistent.js HTTP/1.1`   | `/ecp` 경로 하위의 존재하지 않는 리소스를 요청하여 서버 오류를 유발한다. 서버가 반환하는 오류 페이지 본문에서 후속 요청에 필요한 `ms-ecp-canary` 토큰 값을 추출한다. | [1]  |

### Validation

| 공격 기법                       | 예시                             | 설명                                                                                                                                      | 출처 |
|---------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|------|
| HTTP Response Code Verification | `Response: HTTP/1.1 200 OK`      | Canary가 없는 요청은 HTTP 500 오류를 반환한다는 점에서, 최종 공격 요청에 대해 200 OK와 같은 성공 코드가 반환되는지 확인한다. 이는 서버가 인증 우회를 인지하지 못하고 해당 명령을 성공적으로 처리했음을 의미한다.  | [1]  |

### Exploitation

| 공격 기법                       | 예시                                                                                                                                                          | 설명                                                                                                                             | 출처 |
|---------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|------|
| Inbox Rule Creation via ECP     | `POST /ecp/victim@contoso/.../NewObject?msExchEcpCanary=...`<br>`Cookie: SecurityToken=x`<br>`Body: {"properties":{"RedirectTo":[{"RawIdentity":"attacker@contoso"}]}}` | 인증 우회 후 획득한 `Canary` 토큰과 `SecurityToken` 쿠키를 사용하여, 피해자(`victim@contoso`)의 사서함에 새로운 메일 전달 규칙을 생성하는 POST 요청을 보낸다. `RedirectTo` 파라미터를 통해 모든 수신 메일이 공격자의 주소(`attacker@contoso`)로 리디렉션되도록 설정한다.          | [1]  |

### Persistence

| 공격 기법                     | 예시                                                                                                           | 설명                                                                                                                                                | 출처 |
|-------------------------------|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|------|
| Stealthy Forwarding Rule      | `Set-Mailbox -Identity "victim" -DeliverToMailboxAndForward $true`                                            | 공격자는 탐지를 피하기 위해, 메일 전달 규칙 설정 시 원본 메일이 피해자의 사서함에도 정상적으로 배달되도록 설정할 수 있다. 이를 통해 피해자가 이상 징후를 감지하기 어렵게 만들어 지속성을 유지한다. 공격자는 탐지를 피하기 위해, 메일 전달 규칙 설정 시 원본 메일이 피해자의 사서함에도 정상적으로 배달되도록 설정할 수 있다. 이를 통해 피해자가 이상 징후를 감지하기 어렵게 만들어 지속성을 유지한다.                                                             | [A]  |

### Exfiltration

| 공격 기법                   | 예시                                                            | 설명                                                                                                                           | 출처 |
|-----------------------------|-----------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|------|
| Real-time Email Harvesting  | 메일 전달 규칙을 통해 피해자의 모든 수신 이메일이 공격자 계정으로 실시간 전송됨 | Exploitation 단계에서 설정한 전달 규칙을 통해 피해자의 모든 수신 이메일이 실시간으로 공격자의 계정으로 자동 전송되어 데이터가 외부로 유출된다.                                     | [1]  |

### Cleanup

| 공격 기법               | 예시                                       | 설명                                                                                                                    | 출처 |
|-------------------------|--------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|------|
| Inbox Rule Removal      | `Remove-InboxRule -Identity "victim\RuleName"` | 공격자는 포렌식 분석을 방해하고 자신의 흔적을 은폐하기 위해, Exploitation 단계에서 생성했던 메일 전달 규칙을 `Remove-InboxRule` 명령어를 사용하여 삭제한다.                 | [C]  |

---

## 분류에 해당하는 문서
- [1] [ProxyToken: An Authentication Bypass in Microsoft Exchange Server](https://www.zerodayinitiative.com/blog/2021/8/30/proxytoken-an-authentication-bypass-in-microsoft-exchange-server)

## 기타 참고 문헌
- [A] [Set-Mailbox (ExchangePowerShell)](https://learn.microsoft.com/ko-kr/powershell/module/exchange/set-mailbox?view=exchange-ps)  
- [B] [Exchange Penetration Testing (GitHub)](https://github.com/kh4sh3i/exchange-penetration-testing)  
- [C] [Remove-InboxRule (ExchangePowerShell)](https://learn.microsoft.com/ko-kr/powershell/module/exchange/remove-inboxrule)

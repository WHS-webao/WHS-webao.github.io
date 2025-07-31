---
title: JWT Algorithm Confusion
description: 
published: true
date: 2025-07-30T16:57:45.470Z
tags: http response code verification, public key retrieval, jwt algorithm confusion, public‑key hmac signature forgery, alternate token delivery, privilege escalation via jwt claims, data exfiltration via elevated privileges
editor: markdown
dateCreated: 2025-07-23T11:14:46.660Z
---

# JWT Algorithm Confusion
## 개요
“JWT Algorithm Confusion”는 JWT(JSON Web Token)의 `alg` 헤더에 명시된 서명 알고리즘이 RS256과 HS256처럼 서로 다른 키 체계를 사용하는 경우, 서버가 이를 혼동하여 공격자가 임의로 서명된 토큰을 생성할 수 있는 취약점이다. RS256은 비대칭 키 기반으로 서버는 공개키로 서명을 검증해야 하지만, 검증 로직이 `alg` 값에 따라 HS256으로 오해하고 공개키를 대칭키처럼 재사용하면, 공격자는 공개키를 이용해 HS256 방식의 위조 서명을 생성할 수 있다. 이로 인해 서명이 유효한 것처럼 처리되며, 결과적으로 인증 우회 및 권한 탈취가 발생할 수 있다.


## 분류

> 메타 데이터 해석 불일치 (Metadata Interpretation Gap)  
> → 신원·무결성 해석 불일치 (Identity & Integrity Mismatch)  
> → JWT Algorithm Confusion

## 절차 (HS256<->RS256 Algorithm Confusion 기준)

| Phase            | Description                                                                                                                                                          |
|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Reconnaissance   | 정상 JWT를 탈취해 `alg` 값, `kid` 값을 확인하고, JWKS(JSON Web Key Set) 엔드포인트(`/.well-known/jwks.json`)를 요청하여 해당 `kid`에 매핑되는 공개키를 획득한다.        |
| Divergence       | JWT Header의 `alg`를 `HS256`으로 변경하고 원래 토큰의 `kid`를 서버가 해당 키를 찾게끔 그대로 두거나, 공격자가 얻은 공개키의 `kid`로 덮어쓴다.                                                |
| Mutation         | 획득한 공개키를 HMAC의 비밀키로 사용하여 HS256 서명을 생성하고 새 서명을 Signature 필드에 삽입한다.                                                              |
| Evasion          | 변조된 JWT를 Authorization 헤더가 아닌 Cookie, URL 파라미터 등에 숨기거나 Base64url 변형을 적용해 탐지를 회피한다.                                                   |
| Validation       | 변조 토큰을 포함해 요청을 보내고, HTTP 200 응답 또는 리소스 접근 여부로 우회 성공을 확인한다.                                                                       |
| Exploitation     | 우회 토큰을 사용해 관리자 전용 API 호출, 타 사용자 계정 조작 등 권한 상승 행위를 수행한다.                                                                        |
| Exfiltration     | 획득한 권한으로 사용자 정보, 설정 등 민감 데이터를 탈취하거나 추가 내부 기능을 실행한다.                                                                           |

<div style="text-align: left;">
  <img
    src="/jwt_algorithm_confusion.png"
    alt="JWT Algorithm Confusion 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>
## 공격기법

### Reconnaissance

| 공격 이름       | 예시 페이로드                          | 설명                                                                                                                              | 출처 |
|-----------------|----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|------|
| 공개키 획득     | `GET /.well-known/jwks.json HTTP/1.1`   | 서버의 JWKS(JSON Web Key Set) 엔드포인트에 접근하여 서명 검증에 사용되는 공개키를 획득한다. 이 키는 이후 HS256 서명을 위조하는 데 사용된다. | [B]  |

### Divergence

| 공격 이름            | 예시 페이로드                                                                                                                                                      | 설명                                                                                                                         | 출처      |
|----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|----------|
| 알고리즘 헤더 변경   | ```json<br>// Original Header<br>{"alg":"RS256","typ":"JWT","kid":"..."}<br>// Modified Header<br>{"alg":"HS256","typ":"JWT","kid":"..."}<br>```                     | JWT 헤더의 `alg` 값을 RS256에서 HS256으로 변경한다. 서버가 이 헤더 값을 신뢰하여 검증 로직을 HMAC 방식으로 전환하도록 유도한다. | [1], [B] |

### Mutation

| 공격 이름                      | 예시 페이로드                                                                                                                                                                      | 설명                                                                                                                        | 출처      |
|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|----------|
| 공개키를 이용한 HMAC 서명 생성 | ````text<br>HMAC_SHA256( base64UrlEncode(header) + "." + base64UrlEncode(payload), rsa_public_key )<br>````                                                                              | Reconnaissance 단계에서 획득한 RSA 공개키를 HMAC-SHA256의 비밀 키로 사용하여 새로운 서명을 생성한다. 이 서명은 변조된 토큰의 마지막 부분에 추가된다. | [1], [B] |

### Evasion

| 공격 이름          | 예시 페이로드                                                                          | 설명                                                                                                           | 출처 |
|--------------------|-----------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|------|
| Cookie 전달 우회   | `GET /admin/dashboard HTTP/1.1`<br>`Cookie: authToken=<modified_jwt>`                   | 변조된 JWT를 표준 Authorization 헤더 대신 쿠키에 담아 전송하여, 헤더 기반 보안 탐지 시스템을 우회한다.           | [A]  |
| URL 파라미터 전달  | `GET /resource?token=<modified_jwt>`                                                    | 변조된 JWT를 URL 쿼리 파라미터에 포함시켜 전송한다. 이는 WAF나 API 게이트웨이가 특정 헤더만 검사할 경우 유효한 우회 기법이 될 수 있다. | [A]  |

### Validation

| 공격 이름           | 예시 페이로드                                                                         | 설명                                                                                                                               | 출처 |
|---------------------|----------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|------|
| 보호된 리소스 접근  | `GET /api/admin/dashboard HTTP/1.1`<br>`Authorization: Bearer <modified_jwt>`         | 변조된 토큰으로 관리자 전용 엔드포인트 등 접근이 제한된 리소스에 요청을 보내고, HTTP 200 OK 응답을 통해 공 성공 여부를 확인한다. | [B]  |

### Exploitation

| 공격 이름              | 예시 페이로드                                                                                     | 설명                                                                                                                                 | 출처 |
|------------------------|--------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|------|
| 관리자 권한 클레임 삽입 | ```json<br>{"sub":"user123","role":"admin","iat":1672531200}```                                   | 토큰의 Payload에 `role: "admin"`이나 `isAdmin: true`와 같은 권한 클레임을 추가하여 일반 사용자를 관리자로 위장하고 시스템의 모든 기능에 접근한다. | [C]  |

### Exfiltration

| 공격 이름               | 예시 페이로드                                                                                  | 설명                                                                                                                      | 출처 |
|-------------------------|-----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|------|
| 권한 상승을 통한 데이터 탈취 | `GET /api/v1/users/all HTTP/1.1`<br>`Authorization: Bearer <modified_jwt>`                  | 획득한 관리자 권한을 이용해 전체 사용자 목록, 개인정보, 시스템 설정 등 민감 데이터를 조회하는 API를 호출하여 유출을 완료한다. | [D]  |

---

## 분류에 해당하는 문서
- [1] [JWT Deep Dive into Algorithm Confusion](https://redfoxsec.com/blog/jwt-deep-dive-into-algorithm-confusion/)

## 기타 참고 문헌
- [A] [JSON Web Token for Java Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)  
- [B] [PortSwigger: JWT Algorithm Confusion](https://portswigger.net/web-security/jwt/algorithm-confusion)  
- [C] [Attacking JWT Authentication](https://www.sjoerdlangkemper.nl/2016/09/28/attacking-jwt-authentication/)  
- [D] [OWASP Top 10 A01 2021 – Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)

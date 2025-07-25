---
title: Psychic Signature Attack
description: 
published: true
date: 2025-07-25T16:29:29.010Z
tags: 
editor: markdown
dateCreated: 2025-07-23T11:15:55.723Z
---

# Psychic Signature Attack
## 개요

“Psychic Signature Attack”은 입력 메시지가 전혀 없어도 디지털 서명 검증이 성공하도록 처리하는 검증 로직을 악용해, 메시지 위조와 인증 우회를 가능하게 만드는 공격 기법이다. Java JDK 15\~18 버전에서는 `Signature.verify()` 호출 시 서명 대상 데이터가 생략되어도 검증이 성공하도록 처리 되었으며, 이로 인해 공격자는 메시지 없이도 유효한 서명을 생성하거나 검증을 통과할 수 있다. 이 취약 구조는 서명 검증 로직이 메시지 존재를 전제로 무결성을 보장하는 보안 모델을 근본적으로 무력화하며, 결과적으로 신원 위조 및 인증 우회로 이어질 수 있다.


---

## 분류

>메타데이터 해석 불일치 (Metadata Interpretation Gap)
>→ 신원·무결성 해석 불일치 (Identity & Integrity Mismatch)
>→ Psychic Signature Attack

---

## 절차

| Phase          | Description                                                                                                      |
| -------------- | ---------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 대상 서비스가 Java 15–18을 사용하고 ECDSA-서명 기반 인증(TLS, JWT, WebAuthn…) 을 활용하는지 판별한다.                                       |
| Obfuscation    | `r=0, s=0` 또는 대체 0-값 벡터(Wycheproof test vectors)로 빈 서명을 생성하고, IEEE‑P1363 또는 ASN.1 DER 로 인코딩하여 정상 서명처럼 보이도록 위장한다. |
| Injection      | 위조 서명을 JWT 시그니처, TLS 핸드셰이크, SAML Assertion, WebAuthn 메시지 등에 삽입해 서버에 전송한다.                                        |
| Bypass         | 취약한 Java ECDSA 검증은 `r`, `s` 값이 0인지 확인하지 않아 서명을 무조건 통과시켜 인증, 무결성 검사를 우회한다.                                        |
| Validation     | 서버에서 서명이 유효로 판정되는지 확인한다. `sig.verify(blankSignature) ➜ true` 와 같이 서버에서 서명이 유효로 판정되는지 확인한다. .                     |
| Exploitation   | 공격자는 임의의 신원으로 로그인하거나 TLS MITM으로 통신을 변조하고, 서명 기반 접근 제어를 무력화한다.                                                    |
| Exfiltration   | 얻은 권한으로 민감 데이터 다운로드, 내부 API 호출, 추가 침투 등을 수행한다.                                                                   |

---
<div style="text-align: left;">
  <img
    src="/psychic_signature_attack.png"
    alt="Psychic_Signature_Attack 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>

## 공격 기법

### Reconnaissance

| 공격 이름                                 | 예시                                         | 설명                                                                                                      | 출처               |
| ------------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------- | ---------------- |
| Vulnerable Environment Identification | `java -version → openjdk version "17.0.1"` | 대상 서비스가 ECDSA 기반 인증(TLS, JWT 등)을 사용하며, 서명 검증 로직이 취약한 Java 버전(15, 16, 17, 18)에서 실행되는지 식별하여 공격 가능성을 판별한다. | \[1], \[2], \[4] |
<Br>

### Obfuscation

| 공격 이름                   | 예시         | 설명                                                                                            | 출처         |
| ----------------------- | ---------- | --------------------------------------------------------------------------------------------- | ---------- |
| Blank Signature Forgery | `r=0, s=0` | 서명의 두 구성요소인 `r`과 `s` 값을 모두 0으로 설정하여, 어떤 메시지나 공개키에도 유효한 것으로 위장된 빈 서명(Blank Signature)을 생성한다.   | \[1], \[4] |
| Signature Encoding      | ASN.1 DER  | `r=0, s=0`으로 생성된 서명을 ASN.1 DER 형식으로 인코딩하여, 구문적으로는 유효한 서명 데이터처럼 보이도록 위장하고 파서가 정상적으로 처리하도록 만든다. | \[1], \[4] |

<Br>
  
### Injection

| 공격 이름                   | 예시                                                            | 설명                                                                                      | 출처         |
| ----------------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ---------- |
| Forged JWT Submission   | `Authorization: Bearer [header].[payload].[forged_signature]` | 위조된 빈 서명을 JWT의 서명 필드에 삽입하여, 인증이 필요한 API 엔드포인트에 전송함으로써 서버의 인증 로직을 공격한다.                  | \[1], \[4] |
| Malicious TLS Handshake | ClientKeyExchange 메시지에 위조 서명 포함                               | TLS 핸드셰이크 과정에서 클라이언트 또는 서버가 보내는 서명(예: CertificateVerify)을 위조된 빈 서명으로 대체하여 검증 로직을 무력화한다. | \[1]       |
<Br>

### Bypass

| 공격 이름                     | 예시                          | 설명                                                                                            | 출처                     |
| ------------------------- | --------------------------- | --------------------------------------------------------------------------------------------- | ---------------------- |
| ECDSA Verification Bypass | `r=0, s=0`으로 구성된 DER 인코딩 서명 | 취약한 Java ECDSA 구현체는 서명의 `r`과 `s` 값이 0인지 확인하지 않아, 어떤 메시지에 대해서도 위조된 빈 서명을 유효한 것으로 잘못 판단하고 인증 및 무결성 검증을 완전히 우회한다. | \[1], \[2], \[3], \[4] |
<Br>

### Validation

| 공격 이름                               | 예시                                         | 설명                                                                                         | 출처         |
| ----------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------ | ---------- |
| Authentication Success Confirmation | `Signature.verify(forgedSignature) → true` | 위조된 서명을 검증하는 `Signature.verify()` 메소드가 `true`를 반환하거나, 보호된 리소스에 접근 성공(HTTP 200 OK)하는지 확인한다. | \[1], \[4] |
<Br>

### Exploitation

| 공격 이름                           | 예시                           | 설명                                                                                      | 출처         |
| ------------------------------- | ---------------------------- | --------------------------------------------------------------------------------------- | ---------- |
| Arbitrary Account Takeover      | `{"user": "admin"}` JWT 페이로드 | 서명 검증을 우회하여 JWT 페이로드의 사용자 ID를 `admin` 등 임의의 값으로 조작함으로써, 원하는 사용자의 계정으로 로그인하거나 권한을 상승시킨다. | \[1], \[4] |
| Man-in-the-Middle (MITM) Attack | 위조된 TLS 인증서 사용               | 취약점을 이용해 중간에서 위조된 TLS 서버 인증서에 대한 서명을 생성하고, 클라이언트가 이를 신뢰하게 만들어 통신을 가로채거나 변조한다.           | \[1]       |
| WebAuthn Authentication Bypass  | FIDO2 인증 요청에 위조 서명 사용        | WebAuthn/FIDO2와 같이 강력한 인증 수단에서 사용되는 ECDSA 서명을 위조하여, 패스워드 없이도 다중 요소 인증을 우회하고 계정에 접근한다.   | \[1]       |
<Br>

### Exfiltration

| 공격 이름                  | 예시                     | 설명                                                                                    | 출처   |
| ---------------------- | ---------------------- | ------------------------------------------------------------------------------------- | ---- |
| Privileged Data Access | `GET /api/admin/users` | 성공적으로 탈취한 관리자 권한을 사용하여, 일반 사용자는 접근할 수 없는 API 엔드포인트를 호출하거나 데이터베이스에 직접 접근해 민감 정보를 유출한다. | \[3] |

---

## 분류에 해당하는 문서
- \[1\] Neil Madden, “[CVE-2022-21449: Psychic Signatures in Java](https://neilmadden.blog/2022/04/19/psychic-signatures-in-java/)”, 2022‑04‑19.  
- \[2\] OpenJDK Vulnerability Advisory “[CVE-2022-21449](https://openjdk.org/groups/vulnerability/advisories/2022-04-19)”, 2022‑04‑19.  
- \[3\] NVD Detail — [CVE-2022-21449](https://nvd.nist.gov/vuln/detail/cve-2022-21449)  
- \[4\] Connect2id, “[Java ECDSA CVE-2022-21449 Analysis](https://connect2id.com/blog/cve-2022-21449)”

## 기타 참고 문헌
- \[A\] [GitHub PoC — CVE-2022-21449-TLS-PoC](https://github.com/notkmhn/CVE-2022-21449-TLS-PoC)  
- \[B\] [Oracle CPU April 2022 Advisory](https://www.oracle.com/security-alerts/cpuapr2022.html)  
- \[C\] [Project Wycheproof ECDSA Test Vectors](https://github.com/C2SP/wycheproof)  
- \[D\] [JFrog Blog: Analyzing the New Java Crypto Vulnerability](https://jfrog.com/blog/cve-2022-21449-psychic-signatures-analyzing-the-new-java-crypto-vulnerability/)





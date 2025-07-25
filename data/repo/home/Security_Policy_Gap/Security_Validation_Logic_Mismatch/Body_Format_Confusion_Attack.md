---
title: Body_Format_Confusion_Attack
description: 
published: true
date: 2025-07-25T16:54:07.590Z
tags: 
editor: markdown
dateCreated: 2025-07-22T16:56:31.928Z
---

# Body Format Confusion Attack

## 개요
“Body Format Confusion Attack”는 WAF(Web Application Firewall)와 백엔드 애플리케이션이 동일 HTTP 요청 본문의 Content-Type을 다르게 해석함으로써 발생하는 취약점이다. 공격자는 Content-Type: application/json 또는 application/x-www-form-urlencoded를 명시하면서, 양쪽 형식의 파서가 다르게 처리할 수 있는 본문 구조를 조작해, WAF 우회를 유도한다. 특히 여러 콘텐츠 형식에 대해 복수 파서가 혼합된 환경에서 필드 중복, 구조 중첩, 우선순위 차이 등을 이용해 서버 측 동작을 속일 수 있으며, 결과적으로 인증 우회, 필터링 우회, 원격 명령 실행 등의 공격으로 이어질 수 있다.

## 분류

> 보안 정책 적용 불일치 (Security Policy Gap)  
> → 보안 검증 로직 불일치 (Security Validation Logic Mismatch)  
> → Body Format Confusion Attack (WAFFLED)

## 절차

| Phase            | Description                                                                                                                                                                                        |
|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Reconnaissance   | 서비스 앞단의 WAF 제품(Cloudflare, AWS WAF, Cloud Armor 등)과 백엔드 애플리케이션 프레임워크(Express, Flask, Spring Boot 등)를 식별하고, 양측이 허용하는 Content-Type, 파서 모듈, RFC 버전을 비교해 헤더·바디 파싱 규칙이 엇갈릴 가능성이 있는 지점을 체계적으로 목록화한다. |
| Obfuscation      | URL 인코딩, Zero-Width 문자, 널 바이트, RFC 5987 스타일 파라미터 분할 등을 적용해 변형 패턴이 시그니처나 정적 규칙에 즉시 포착되지 않도록 난독화하고, 필요 시 링크 단축·암호화 채널을 통해 전달해 탐지·로그 상 노이즈를 최소화한다.                                              |
| Manipulation     | 준비한 우회용 요청을 실제로 전송해 WAF가 잘못 분할·해석하도록 유도하여 보안 규칙 적용을 우회하고, 동시에 백엔드 파서가 요청을 정상 해석해 XSS·SQL 주입·명령 실행 페이로드가 그대로 수행되도록 만든다.                                                                    |
| Validation       | 우회 성공 여부를 확인하기 위해 서버 측에서 수신 바디를 별도 파일로 저장하거나, 응답 값·로그 차이를 자동으로 비교해 변조가 통과했는지 판별하고, 필요 시 재조합된 페이로드를 반복적으로 테스트해서 신뢰도 높은 변형을 추려낸다.                                                      |
| Exploitation     | 검증을 통과한 요청을 이용해 세션 토큰 탈취, 원격 코드 실행, 캐시 포이즈닝, 내부 API 호출 등 실제 비즈니스 영향을 주는 공격 시나리오를 실행하며, 추가 권한 상승이나 가용성 저해까지 노린다.                                                                          |
| Exfiltration     | 획득한 쿠키·자격 증명·시크릿 키 등을 fetch·Beacon API·WebSocket·DNS exfil 같은 경로로 공격자 서버에 전송해서 장기적으로 활용하거나 후속 침투를 위한 발판을 마련한다.                                                                          |
<div style="text-align: left;">
  <img
    src="/body_format_confusion_attack.png"
    alt="Body Format Confusion Attack 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>

## 공격 기법

### Reconnaissance

| 공격 이름                             | 예시 페이로드                                                                                                        | 설명                                                                                                                                                                     | 출처 |
|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| Content-Type Interchangeability Survey | Burp “Change body encoding”으로 application/x-www-form-urlencoded → multipart/form-data 전환 후 응답 동일성 확인 | 응답이 동일하면 애플리케이션이 두 포맷을 구분하지 않음을 식별해 우회 대상 조합을 선정한다.                                                                                  | [1]  |
| Parse-Log Verification Setup           | 프레임워크가 파싱한 바디를 로그/파일에 기록하도록 구성                                                               | 로그에 공격 페이로드가 그대로 존재하면 해당 요청이 우회 성공했음을 판단할 근거가 된다.                                                                                      | [1]  |

### Obfuscation

| 공격 이름                              | 예시 페이로드                                                                                           | 설명                                                                                                                                                                    | 출처 |
|----------------------------------------|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|
| Whitespace Alteration                  | `Content-Type:\t multipart/form-data; boundary*0=re;boundary*1=al`                                       | 공백/탭 치환으로 시그니처를 흐리고, 파서는 RFC 2231 연속 파라미터를 이어붙여 해석한다.                                                                                   | [1]  |
| Header Separator Manipulation in Body  | `content-disposition: form-data; name="f1" \x00`                                                        | 본문 헤더 구분자에 비표준 문자를 넣어 WAF 토큰화를 깨지만 백엔드 파서는 허용한다.                                                                                         | [1]  |
| Disrupted Body Field                   | `content-disposition: form-data; name="field1 \x00 "`                                                    | 필드명에 불법 문자를 삽입해 필터 정규식을 무력화하고 파서만 정상 처리하게 만든다.                                                                                         | [1]  |
| Field Wrapper Manipulation (JSON)      | `{ "field1" \x00 : "<script>alert(document.cookie)</script>" }`                                           | JSON 필드명과 콜론 사이에 문자를 삽입해 필드 파싱을 교란하고 필터를 회피한다.                                                                                             | [1]  |

### Manipulation

| 공격 이름                         | 예시 페이로드                  | 설명                                                                                                                                             | 출처 |
|-----------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|------|
| Boundary Delimiter Manipulation   | `\r\n—boundary` 시퀀스 변경    | 경계 앞 CRLF를 제거/변형해 WAF 파싱을 혼란시키고 프레임워크는 정상 경계를 인식하게 한다.                                                           | [1]  |
| Content-Type Parameter Tweak      | `C o ntent-Type: multipart/form-data;` | 	`Content‑Type` 헤더 이름에 공백·문자 삽입·삭제로 WAF 필터 정규식과 일치하지 않도록 조작하고, 백엔드 파서는 정상적으로 인식하도록 만든다.                                                               | [1]  |
| Distorted Header Injection to Body | `conten\x00-extra: something` | 본문에 헤더 구문처럼 보이는 제어 문자를 삽입해 WAF가 헤더 블록으로 인식하지 못하게 하고, 백엔드 파서는 이를 데이터로 처리하게 만든다.                                                                    | [1]  |
| DOCTYPE Closure Confusion (XML)   | `<!DOCTYPE BOOK [...] ] </BOOK>` | XML DOCTYPE 종료를 교란해 WAF 파싱을 실패시키고 프레임워크는 정상 처리하게 한다.                                                                  | [1]  |
| Boundary Continuation Abuse       | `boundary=fake-boundary;boundary*0=real-;boundary*1=boundary` | WAF는 첫 boundary만, 프레임워크는 RFC 2231 연속 파라미터를 이어붙여 실제 경계를 사용하게 만든다.                                                      | [1]  |

### Validation

| 공격 이름                   | 예시 페이로드                                 | 설명                                                                                                                                   | 출처 |
|-----------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|------|
| Stored-Payload Presence Check | 서버 저장 로그에서 `<script>alert(document.cookie)</script>` 존재 여부 확인 | 파서 결과에 페이로드가 남아 있으면 WAF 우회가 성공한 것으로 확정한다.                                                                    | [1]  |
| Response Parity Test         | 변형 전/후 요청의 상태코드·본문 비교          | 응답이 동일하면 변형이 애플리케이션에서도 동일하게 해석됐음을 의미한다.                                                                   | [1]  |

### Exploitation

| 공격 이름                    | 예시 페이로드                                        | 설명                                                                                                                                     | 출처 |
|------------------------------|-----------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|------|
| Deliver Unmodified Attack Payload | Listing 3의 `<script>alert(document.cookie)</script>` 유지 | 페이로드는 변형하지 않고 바디/헤더만 조작해 WAF만 속이고 앱에서 그대로 실행되게 한다.                                                         | [1]  |
| Execute Payload After Bypass    | 우회된 요청으로 XSS 실행 확인                         | 파싱 불일치로 통과된 악성 본문이 실제로 애플리케이션에서 처리·실행된다.                                                                     | [1]  |

---

## 분류에 해당하는 문서
- [1] https://arxiv.org/pdf/2503.10846v1

---
title: Semantic Gap
description: 
published: true
date: 2025-07-15T08:40:33.718Z
tags: 
editor: markdown
dateCreated: 2025-07-12T21:16:08.809Z
---

# 프로젝트명: 가지마 웹바오
## 1. 구문 해석 불일치  
_동일한 입력(요청/데이터)을 서로 다른 파서가 다르게 해석해 발생하는 갭_

- **문제 정의**  
  - 예: HTTP 메시지 파싱, URL·헤더 필드 파싱, HTML 구문 파싱 등
- **발견된 취약점/사례**  
  - 예: Request Smuggling, Header Parsing Discrepancy, HTML Sanitizer Bypass
- **적용된 분석 기법**  
  - 예: Differential Fuzzing, MutaGen, 로직 리버스엔지니어링
- **결과/성과**  
  - CVE 등록/제보, PoC 개발, WAF 우회 패턴 식별  

---

## 2. 컨텍스트 경계 불일치  
_컴포넌트 간 컨텍스트 전환(프레임워크→라이브러리→브라우저 등) 시 경계가 엇갈려 발생하는 갭_

- **문제 정의**  
  - 예: Same-Origin vs. Signed HTTP Exchange, Server Push vs. CSP 경계
- **발견된 취약점/사례**  
  - 예: Cookie Tossing, Subdomain Session Fusion
- **적용된 분석 기법**  
  - 예: 프로토콜 트레이싱, 전문 디버깅 툴 활용
- **결과/성과**  
  - 권한 모델 개선 제안, SameSite 정책 강화  

---

## 3. 데이터 표현 모델 불일치  
_JSON↔XML, 바이너리↔텍스트 등 동일 데이터의 서로 다른 표현 간 차이를 이용한 갭_

- **문제 정의**  
  - 예: JSON Path vs. XML XPath, multipart vs. urlencoded
- **발견된 취약점/사례**  
  - 예: XML Injection via Encoding Mismatch, Parameter Pollution
- **적용된 분석 기법**  
  - 예: 포맷 변환 테스트, 언어별 파서 비교
- **결과/성과**  
  - 포맷 통합 라이브러리 제안, 보안 가이드 작성  

---

## 4. 메타데이터 해석 불일치  
_헤더·쿼리·캐시 키·Vary·Cache-Control 같은 메타데이터를 구성 요소별로 다르게 해석해 발생하는 갭_

- **문제 정의**  
  - 예: Web Cache Poisoning, Content-Type Confusion, CORS 해석 차이
- **발견된 취약점/사례**  
  - 예: “Gotta cache ’em all…”, Signed HTTP Exchange vs. HTTP/2 Push
- **적용된 분석 기법**  
  - 예: HTTP 헤더 분석, CDN ↔ 앱 서버 테스트
- **결과/성과**  
  - 메타데이터 정규화 프록시(HTTP-Normalizer) 제작, 가이드라인 배포  

---

## 5. 보안 정책 적용 불일치  
_인증·권한·ACL·CSP·Helmet 등 보안 설정이 실제 런타임에 다르게 적용돼 발생하는 갭_

- **문제 정의**  
  - 예: 경로 패턴 매칭 불일치, 헤더 기반 접근 제어 우회
- **발견된 취약점/사례**  
  - 예: Spring WebFlux CVE-2023-34034, Consul L7 Intentions 우회
- **적용된 분석 기법**  
  - 예: 보안 설정 코드 리뷰, 런타임 트레이스·디버깅
- **결과/성과**  
  - 보안 정책 라이브러리 패치, 설정 템플릿 제공  

---

## 부록  
- **참고 자료 & 링크**  
- **용어 정리**  
- **추가 제안 사항**  

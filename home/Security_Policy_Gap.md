---
title: Security Policy Gap
description: 
published: true
date: 2025-07-29T17:14:06.461Z
tags: 
editor: markdown
dateCreated: 2025-07-15T08:30:06.234Z
---

# 보안 정책 적용 불일치 (Security Policy Gap)

## 개요

“보안 정책 적용 불일치”는 요청이 통과하는 경로마다 보안 정책의 해석・우선순위・적용 범위가 달라져, 하나의 행위가 단계별로 서로 다른 허용・차단 결정을 받는 불일치를 말한다. 공격자는 규칙이 느슨한 지점을 찾아 입력 형식・프로토콜 흐름・헤더 구성을 교묘히 바꿔 보안 정책 판단의 혼동을 유발하고, 최종 처리 단계에서 의도한 명령・세션・권한이 그대로 실행되도록 유도해 인증 우회, 정책 무력화, 민감 데이터 노출 같은 중대한 보안 영향을 초래한다.

## 하위 항목

* [보안 검증 로직 불일치 (Security Validation Logic Mismatch)](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Security_Validation_Logic_Mismatch)
* [클라이언트 보안 정책 불일치 (Client-Side Security Policy Mismatch)](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Client-Side_Security_Policy_Mismatch)
* [네트워크 계층 정책 불일치 (Cross-Protocol Policy Mismatch)](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Cross-Protocol_Policy_Mismatch)

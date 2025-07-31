---
title: Security_Validation_Logic_Mismatch
description: 
published: true
date: 2025-07-29T16:12:05.343Z
tags: 
editor: markdown
dateCreated: 2025-07-20T11:54:33.620Z
---

# 보안 검증 로직 불일치 (Security Validation Logic Mismatch)

## 개요

“보안 검증 로직 불일치”는 요청이 통과하는 각 계층(프런트 필터・애플리케이션 등)이 서로 다른 검증 로직과 입력 가정을 적용해 동일 데이터를 상반된 의미로 해석하면서 생기는 취약점이다. 전단 로직은 제한된 규칙 집합으로 안전하다고 판단하지만, 후단 로직은 같은 입력을 별개 규칙이나 완화된 절차로 처리해 검증 연속이 끊어진다. 이러한 검증 로직 불일치는 인증・인가・정책을 우회하게 만들고, 최종적으로 비인가 기능 실행・권한 상승・데이터 노출과 같은 중대한 보안 영향을 초래한다.

## 분류

보안 정책 적용 불일치 (Security Policy Gap) - 보안 검증 로직 불일치 (Security Validation Logic Mismatch)

## 하위 항목

* [Body Format Confusion Attack](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Security_Validation_Logic_Mismatch/Body_Format_Confusion_Attack)
* [Method Override Bypass](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Security_Validation_Logic_Mismatch/Method_Override_Bypass)

---
title: Syntax Parsing Gap
description: 
published: true
date: 2025-07-30T16:31:37.883Z
tags: 
editor: markdown
dateCreated: 2025-07-14T07:47:15.895Z
---

# Syntax Parsing Gap (구문 분석 불일치)

## 개요

“Syntax Parsing Gap (구문 분석 불일치)”은 요청・응답 데이터의 구문을 다루는 파서들이 키 중복 처리, 경계 인식, 경로 정규화, 계층 구조 해석과 같은 세부 규칙을 서로 다르게 적용해, 동일 바이트열이 처리 단계마다 서로 다른 논리 구조로 변환되는 현상이다. 이 간극을 악용한 입력은 앞단 검증을 무해하게 통과하면서 후단 로직에서는 의도한 구조・명령으로 해석돼, 최종적으로 인증 우회・코드 주입・권한 상승 같은 심각한 보안 영향을 초래한다.

## 하위 항목

* [Duplicate Key Handling Mismatch (중복 키 처리 불일치)](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Duplicate_Key_Handling_Mismatch)
* [Boundary Parsing Mismatch (데이터 경계 파싱 불일치)](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Boundary_Parsing_Mismatch)
* [Path Parsing Mismatch (경로 파싱 불일치)](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Path_Parsing_Mismatch)
* [Structural Interpretation Mismatch (구조 해석 불일치)](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Structural_Interpretation_Mismatch)

---
title: Structural_Interpretation_Mismatch
description: 
published: true
date: 2025-07-29T16:02:54.939Z
tags: 
editor: markdown
dateCreated: 2025-07-20T11:23:14.631Z
---

# 구조 해석 불일치 (Structural Interpretation Mismatch)

## 개요

“구조 해석 불일치”는 XML・DOM・SQL・PDF처럼 계층 또는 블록 구조를 가진 데이터를 두 단계(예: 검증 단계, 실행 단계)에서 다른 규칙으로 해석할 때 생긴다. 공격자는 노드 재배치, 중첩 래퍼(wrapper), 네임스페이스 교란, 이중 주석 삽입 등으로 검증 시점에는 안전한 구조처럼 보이도록 꾸미지만, 실행 시점 파서는 이를 새로운 부모・형제 관계나 별도 블록으로 해석해 보호된 노드에 접근하거나 추가 코드를 실행한다. 이 차이로 서명 우회, 필터 우회, 권한 상승 등 다양한 후속 공격이 가능하다.

## 분류

구문 분석 불일치 (Syntax Parsing Gap) - 구조 해석 불일치 (Structural Interpretation Mismatch)

## 하위 항목

* [SQL Smuggling](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Structural_Interpretation_Mismatch/SQL_Smuggling)
* [XML Signature Wrapping](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Structural_Interpretation_Mismatch/XML_Signature_Wrapping)
* [DOMPurify namespace confusion](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Structural_Interpretation_Mismatch/DOMPurify_namespace_confusion)
* [Document Format Parsing Discrepancy](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Structural_Interpretation_Mismatch/Document_Format_Parsing_Discrepancy)
* [Apache Handler Confusion](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Structural_Interpretation_Mismatch/Apache_Handler_Confusion)
* [JSON Structure Confusion](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Structural_Interpretation_Mismatch/JSON_Structure_Confusion)
* [Double Dash SQL Injection](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Structural_Interpretation_Mismatch/Double_Dash_SQL_Injection)
* [Shellshock](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Structural_Interpretation_Mismatch/Shellshock)

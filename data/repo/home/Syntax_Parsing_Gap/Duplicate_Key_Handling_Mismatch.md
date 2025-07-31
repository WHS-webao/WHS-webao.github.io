---
title: Duplicate Key Handling Mismatch
description: 
published: true
date: 2025-07-30T16:32:45.255Z
tags: 
editor: markdown
dateCreated: 2025-07-23T10:52:44.042Z
---

# Duplicate Key Handling Mismatch (중복 키 처리 불일치)

## 개요
“Duplicate Key Handling Mismatch (중복 키 처리 불일치)”는 하나의 요청・파일・메시지에 동일한 키(필드・헤더・프로퍼티)가 두 번 이상 포함될 때, 앞단 컴포넌트와 뒷단 파서가 어느 값을 적용할지 규칙이 다르기 때문에 검증 단계와 실제 실행 단계가 달라지는 현상이다. 예컨데 프록시・WAF는 첫 번째 값을 기준으로 접근 제어를 수행하지만, 애플리케이션은 마지막 값을 채택하거나 모든 값을 병합해 처리함으로써 권한・세션・캐시・설정이 뒤바뀌어 인증 우회, 권한 상승, 데이터 노출로 이어진다.

## 분류
Syntax Parsing Gap (구문 분석 불일치) - Duplicate Key Handling Mismatch (중복 키 처리 불일치)

## 하위 항목

* [HTTP Header Duplication](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Duplicate_Key_Handling_Mismatch/HTTP_Header_Duplication)
* [YAML Key Duplication](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Duplicate_Key_Handling_Mismatch/YAML_Key_Duplication)
* [HTTP Parameter Pollution](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Duplicate_Key_Handling_Mismatch/HTTP_Parameter_Pollution)
* [JSON Key Duplication](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Duplicate_Key_Handling_Mismatch/JSON_Key_Duplication)

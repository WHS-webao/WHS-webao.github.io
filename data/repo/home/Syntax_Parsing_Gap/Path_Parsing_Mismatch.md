---
title: Path_Parsing_Mismatch
description: 
published: true
date: 2025-07-31T16:12:11.551Z
tags: 
editor: markdown
dateCreated: 2025-07-20T11:20:25.192Z
---

# 경로 파싱 불일치 (Path Parsing Mismatch)

## 개요

“경로 파싱 불일치”는 하나의 경로 문자열을 두 구성요소가 서로 다른 정규화・디코딩 규칙으로 해석해, 검증 단계에서 허용된 요청이 실행 단계에서는 전혀 다른 위치나 리소스로 바뀌는 현상이다. 이 차이를 노린 요청은 보안 필터를 무사히 통과하면서 실제로는 보호 디렉터리 밖 파일에 접근하거나 인증이 필요한 엔드포인트로 이동하도록 해석되어, 인증 우회, 민감 정보 노출, 리소스 조작 같은 후속 공격을 유발한다.

## 분류

구문 분석 불일치 (Syntax Parsing Gap) - 경로 파싱 불일치 (Path Parsing Mismatch)

## 하위 항목

* [Cookieless Session Authentication Bypass](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Path_Parsing_Mismatch/Cookieless_Session_Authentication_Bypass)
* [Apache DocumentRoot Confusion](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Path_Parsing_Mismatch/Apache_DocumentRoot_Confusion)
* [Path Traversal Encoding Bypass](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Path_Parsing_Mismatch/Path_Traversal_Encoding_Bypass)
* [Apache Filename Confusion](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Path_Parsing_Mismatch/Apache_Filename_Confusion)
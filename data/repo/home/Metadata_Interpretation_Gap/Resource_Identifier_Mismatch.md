---
title: Resource_Identifier_Mismatch
description: 
published: true
date: 2025-07-29T17:02:17.508Z
tags: 
editor: markdown
dateCreated: 2025-07-20T12:10:31.635Z
---

# 리소스 식별자 해석 불일치 (Resource Identifier Mismatch)

## 개요

“리소스 식별자 해석 불일치”는 시스템이 참조하는 리소스 식별 메타데이터(파일 경로, 패키지・모듈 이름, 도메인・CNAME 별칭, 바이너리 태그 등)를 구성 요소마다 우선순위・정규화 규칙을 달리 적용해, 동일 식별자가 각 단계에서 서로 다른 실제 리소스로 해석되는 불일치다. 이 간극으로 인해 검증 단계가 안전하다고 판단한 메타데이터가 로딩 단계에선 전혀 다른 파일・서버・패키지로 매핑돼, 의도치 않은 코드 실행, 외부 서비스 점유, 공급망 오염 같은 심각한 보안 피해가 발생할 수 있다.

## 분류

메타데이터 해석 불일치 (Metadata Interpretation Gap) - 리소스 식별자 해석 불일치 (Resource Identifier Mismatch)

## 하위 항목

* [YAML Binary Tag Handling](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Resource_Identifier_Mismatch/YAML_Binary_Tag_Handling)
* [Package Source Confusion Attack](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Resource_Identifier_Mismatch/Package_Source_Confusion_Attack)
* [Relative Path Overwrite Attack](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Resource_Identifier_Mismatch/Relative_Path_Overwrite_Attack)
* [Dangling CNAME Takeover](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Resource_Identifier_Mismatch/Dangling_CNAME_Takeover)

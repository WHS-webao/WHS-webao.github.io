---
title: Data Representation Model Gap
description: 
published: true
date: 2025-07-29T17:12:17.244Z
tags: 
editor: markdown
dateCreated: 2025-07-15T08:31:38.581Z
---

# 데이터 표현 모델 불일치 (Data Representation Model Gap)

## 개요

“데이터 표현 모델 불일치”는 동일한 데이터가 문자 인코딩, 정규화, 이스케이프, 유사문자 치환 등 표현 방식에 따라 파서・플랫폼・보안 로직마다 서로 다른 값・의미로 해석되면서 생기는 불일치를 뜻한다. 공격자는 한 입력을 다중 인코딩하거나 유사문자로 변형해 검사 단계에서 무해하게 통과시킨 뒤, 실행 단계에서 원래 명령・식별자로 재해석되도록 유도하여 계정 스푸핑, 필터 우회, 코드 실행 등 심각한 보안 영향을 일으킬 수 있다.

## 하위 항목

* [유사문자 및 정규화 처리 문제 (Homoglyph & Normalization Mismatch)](https://semanticgap.mjsec.kr/en/home/Data_Representation_Model_Gap/Homoglyph_%26_Normalization_Mismatch)
* [인코딩 및 이스케이프 처리 불일치 (Encoding & Escape Handling Mismatch)](https://semanticgap.mjsec.kr/en/home/Data_Representation_Model_Gap/Encoding_%26_Escape_Handling_Mismatch)

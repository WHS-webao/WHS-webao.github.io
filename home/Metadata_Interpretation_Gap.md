---
title: Metadata Interpretation Gap
description: 
published: true
date: 2025-07-29T17:15:04.564Z
tags: 
editor: markdown
dateCreated: 2025-07-15T08:33:22.225Z
---

# 메타데이터 해석 불일치 (Metadata Interpretation Gap)

## 개요

“메타데이터 해석 불일치”는 메시지에 실린 부가 정보(헤더・토큰・서명・쿠키 등 메타데이터)를 시스템 단계마다 서로 다른 검증 범위・우선순위로 해석해, 동일 요청이 상반된 신뢰 수준・대상・세션으로 처리되는 불일치를 뜻한다. 공격자는 이 간극을 이용해 하나의 메타데이터 값을 이중 삽입・변형해 일부 계층을 속이고, 다른 계층에선 의도한 값이 적용되도록 유도함으로써 인증 우회・권한 상승・데이터 위변조 같은 심각한 보안 문제를 일으킨다.

## 하위 항목

* [신원·무결성 해석 불일치 (Identity & Integrity Mismatch)](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Identity_%26_Integrity_Mismatch)
* [상태·세션 해석 불일치 (State & Session Mismatch)](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/State_%26_Session_Mismatch)
* [리소스 식별자 해석 불일치 (Resource Identifier Mismatch)](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Resource_Identifier_Mismatch)

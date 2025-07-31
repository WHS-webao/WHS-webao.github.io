---
title: Identity_&_Integrity_Mismatch
description: 
published: true
date: 2025-07-31T15:15:35.981Z
tags: 
editor: markdown
dateCreated: 2025-07-20T12:07:51.033Z
---

# 신원·무결성 해석 불일치 (Identity & Integrity Mismatch)

## 개요

“신원・무결성 해석 불일치”는 메시지의 신원・무결성을 보장하는 토큰・서명・헤더・해시 등의 메타데이터가 구성요소마다 검증 범위・우선순위・적용 경계를 달리 해석해, 동일 요청을 상반된 신뢰 수준으로 처리하도록 만드는 불일치를 이용한다. 공격자는 알고리즘・키 우선순위 혼동, 중첩 서명・헤더 삽입, 프로토콜 경계 재작성 등으로 검증 단계를 분기시켜 일부 보호 레이어를 우회하거나 무력화하며, 그 결과 인증 우회・권한 상승・데이터 위변조 같은 중대한 보안 위협을 가할 수 있다.

## 분류

메타데이터 해석 불일치 (Metadata Interpretation Gap) - 신원·무결성 해석 불일치 (Identity & Integrity Mismatch)

## 하위 항목

* [Authentication Bypass via Header Manipulation](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Identity_%26_Integrity_Mismatch/Authentication_Bypass_via_Header_Manipulation)
* [JWT Algorithm Confusion](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Identity_%26_Integrity_Mismatch/JWT_Algorithm_Confusion)
* [JWT None Algorithm Attack](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Identity_%26_Integrity_Mismatch/JWT_None_Algorithm_Attack)
* [Psychic Signature Attack](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Identity_%26_Integrity_Mismatch/Psychic_Signature_Attack)
* [HTTPoxy Header Injection](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/Identity_%26_Integrity_Mismatch/HTTPoxy_Header_Injection)

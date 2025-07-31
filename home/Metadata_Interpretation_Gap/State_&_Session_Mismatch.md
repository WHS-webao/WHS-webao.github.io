---
title: State_&_Session_Mismatch
description: 
published: true
date: 2025-07-31T15:20:04.523Z
tags: 
editor: markdown
dateCreated: 2025-07-20T12:09:19.578Z
---

# 상태·세션 해석 불일치 (State & Session Mismatch)

## 개요

“상태・세션 해석 불일치”는 쿠키 속성・세션 ID・캐시 헤더 등 상태 관리용 메타데이터가 처리 단계마다 만료 시점・우선순위・결합 규칙을 다르게 해석해, 동일 요청임에도 각 계층이 서로 다른 세션, 상태를 참조하게 되는 불일치다. 공격자는 이 간극을 악용하여 보안 계층을 통과한 후 후단 처리 단계에서 외부 세션이 재활용되거나 만료된 상태가 부활해, 잘못된 사용자로 동작이 수행되고 민감 데이터가 노출되는 심각한 위험을 발생시킬 수 있다.

## 분류

메타데이터 해석 불일치 (Metadata Interpretation Gap) - 상태·세션 해석 불일치 (State & Session Mismatch)

## 하위 항목

* [Web Cache Poisoning](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/State_%26_Session_Mismatch/Web_Cache_Poisoning)
* [Web Cache Deception](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/State_%26_Session_Mismatch/Web_Cache_Deception)
* [Cookie Tossing](https://semanticgap.mjsec.kr/en/home/Metadata_Interpretation_Gap/State_%26_Session_Mismatch/Cookie_Tossing)

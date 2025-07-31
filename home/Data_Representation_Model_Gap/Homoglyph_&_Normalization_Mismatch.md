---
title: Homoglyph & Normalization Mismatch
description: 
published: true
date: 2025-07-29T16:48:49.565Z
tags: 
editor: markdown
dateCreated: 2025-07-23T11:11:06.305Z
---

# 유사문자 및 정규화 처리 문제 (Homoglyph & Normalization Mismatch)

## 개요

“유사문자 및 정규화 처리 문제”는 시스템마다 서로 다른 유니코드 정규화 규칙(NFC・NFD 등)과 문자 분류・대소문자 매핑을 적용해, 동일해 보이거나 시각적으로 구분하기 어려운 문자열이 필터링・인증・라우팅 단계마다 다른 식별자로 해석되는 불일치를 다룬다. 공격자는 이 간극을 이용해 도메인・사용자명・이벤트 핸들러・경로 등의 검사 로직을 우회하거나 실제 대상과 구별되지 않는 위장 식별자를 생성할 수 있으며, 그 결과 계정 탈취, 스푸핑, 정책 우회 같은 심각한 보안 피해로 이어진다.

## 분류

데이터 표현 모델 불일치(Data_Representation_Model_Gap) - 유사문자 및 정규화 처리 문제 (Homoglyph & Normalization Mismatch)

## 하위 항목

* [Unicode Normalization Bypass](https://semanticgap.mjsec.kr/en/home/Data_Representation_Model_Gap/Homoglyph_&_Normalization_Mismatch/Unicode_Normalization_Bypass)

* [IDN Homograph Spoofing](https://semanticgap.mjsec.kr/en/home/Data_Representation_Model_Gap/Homoglyph_&_Normalization_Mismatch/IDN_Homograph_Spoofing)
* [JavaScript Event Normalization Bypass](https://semanticgap.mjsec.kr/en/home/Data_Representation_Model_Gap/Homoglyph_&_Normalization_Mismatch/JavaScript_Event_Normalization_Bypass)
* [Homoglyph Username Bypass](https://semanticgap.mjsec.kr/en/home/Data_Representation_Model_Gap/Homoglyph_&_Normalization_Mismatch/Homoglyph_Username_Bypass)

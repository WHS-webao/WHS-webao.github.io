---
title: Visual Representation Mismatch
description: 
published: true
date: 2025-07-29T16:08:17.865Z
tags: 
editor: markdown
dateCreated: 2025-07-22T16:39:00.479Z
---

# 시각적 표현 불일치 (Visual Representation Mismatch)

## 개요

“시각적 표현 불일치”는 브라우저・모바일 UI에서 요소의 시각적 배치(투명도・레이어・스크롤 오프셋 등)와 실제 클릭・터치가 전달되는 상호작용 경계가 어긋나도록 조작해, 사용자가 보지 못하거나 의도하지 않은 영역에 입력이 전달되는 취약점이다. 레이아웃 엔진과 이벤트 라우팅 로직의 처리 차이를 악용해, 공격자는 겹쳐진 iframe・div・canvas 등을 투명・오프스크린 상태로 배치해 보안 경계를 넘어선 행위를 유도할 수 있다. 그 결과 사용자 클릭이 보이지 않는 버튼, 권한 승인 대화상자, 혹은 외부 사이트로 리디렉션되는 링크에 전달돼 인증 우회, 악성 기능 활성화, 세션 탈취 등으로 이어진다.

## 분류

인지 맥락 불일치 (Perceptual Context Gap) - 시각적 표현 불일치 (Visual Representation Mismatch)

## 하위 항목

* [Double Clickjacking](https://semanticgap.mjsec.kr/en/home/Perceptual_Context_Gap/Visual_Representation_Mismatch/Double_Clickjacking)
* [Document Format Parsing Discrepancy](https://semanticgap.mjsec.kr/en/home/Perceptual_Context_Gap/Visual_Representation_Mismatch/Document_Format_Parsing_Discrepancy)

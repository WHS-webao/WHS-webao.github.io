---
title: Browser Tab Context Mismatch
description: 
published: true
date: 2025-07-29T16:05:48.048Z
tags: 
editor: markdown
dateCreated: 2025-07-20T11:36:24.259Z
---

# 브라우저 탭 컨텍스트 불일치 (Browser Tab Context Mismatch)

## 개요

“브라우저 탭 컨텍스트 불일치”는 브라우저가 새 창을 열거나 탭을 전환할 때 어느 탭이 어느 사이트와 연결돼 있는지에 대한 기술적 컨텍스트와 사용자의 인지 컨텍스트가 엇갈리는 현상이다. 한쪽 탭의 스크립트가 다른 탭을 열거나 제어하면서 주소 표시줄·UI 타이틀 등 시각 정보는 거의 변하지 않아도, 실제 보안 경계(출처·세션·페이지 상태)는 이미 달라질 수 있다. 이 불일치는 피싱, cross-site 요청, 세션 재사용 등 다양한 탭 간 공격의 기반이 된다.

## 분류

인지 맥락 불일치 (Perceptual Context Gap) - 브라우저 탭 컨텍스트 불일치 (Browser Tab Context Mismatch)

## 하위 문서

* [Reverse Tabnabbing](https://semanticgap.mjsec.kr/en/home/Perceptual_Context_Gap/Browser_Tab_Context_Mismatch/Reverse_Tabnabbing)
* [Tabnabbing](https://semanticgap.mjsec.kr/en/home/Perceptual_Context_Gap/Browser_Tab_Context_Mismatch/Tabnabbing)

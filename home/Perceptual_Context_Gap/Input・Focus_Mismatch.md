---
title: Input/Focus Mismatch
description: 
published: true
date: 2025-07-29T16:10:43.729Z
tags: 
editor: markdown
dateCreated: 2025-07-22T16:48:38.457Z
---

# 포커스 및 입력 컨텍스트 불일치 (Input・Focus Mismatch)

## 개요

“포커스 및 입력 컨텍스트 불일치”는 브라우저・OS의 포커스・입력 이벤트 전달 체계와 사용자가 인식하는 대상 요소 간의 경계가 어긋나도록 조작해, 키 입력・클립보드・파일 드래그 등 실제 인터렉션이 공격자가 숨겨 둔 컴포넌트에 전달되게 만드는 기법이다. 공격자는 투명・오프스크린 입력창, CSS tabindex・pointer-events 조작, 클립보드 API 호출 등을 활용해 화면상 보이지 않는 폼・버튼・URL로 포커스를 유도하거나 입력 스트림을 가로채어, 결과적으로 사용자가 의도치 않은 인증 토큰 제출, 악성 명령 실행, 민감 데이터 유출이 발생 가능하다.

## 분류

인지 맥락 불일치 (Perceptual Context Gap) - 포커스 및 입력 컨텍스트 불일치 (Input・Focus Mismatch)

## 하위 항목

* [Clipjacking](https://semanticgap.mjsec.kr/en/home/Perceptual_Context_Gap/Input・Focus_Mismatch/Clipjacking)

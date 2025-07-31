---
title: Encoding & Escape Handling Issues
description: 
published: true
date: 2025-07-29T16:58:04.756Z
tags: 
editor: markdown
dateCreated: 2025-07-23T11:11:40.376Z
---

# 인코딩 및 이스케이프 처리 불일치 (Encoding & Escape Handling Mismatch)

## 개요

“인코딩 및 이스케이프 처리 불일치”는 입력 데이터가 여러 인코딩(UTF-8・UTF-16・Shift\_JIS 등)과 이스케이프 방식(퍼센트・역슬래시・HTML 엔티티 등)을 혼합해 표현될 때, 시스템 구성요소마다 정규화・디코딩 순서와 범위가 달라 동일 바이트열을 서로 다른 문자・제어 시퀀스로 해석하는 불일치를 뜻한다. 웹 서버・DB・브라우저・CLI 등 파서 체계가 중첩된 인코딩을 일부만 해제하거나 재차 인코딩하면서 필터링 로직과 실행 로직 사이 의미가 뒤틀려, 최종적으로 인가 우회, 스크립트 실행, 경로 변조 등 치명적 보안 영향을 일으킨다.

## 분류

데이터 표현 모델 불일치 (Data Representation Model Gap) - 인코딩 및 이스케이프 처리 문제 (Encoding & Escape Handling Mismatch)

## 하위 항목

* [Charset Encoding Confusion Attack](https://semanticgap.mjsec.kr/en/home/Data_Representation_Model_Gap/Encoding_&_Escape_Handling_Mismatch/Charset_Encoding_Confusion_Attack)

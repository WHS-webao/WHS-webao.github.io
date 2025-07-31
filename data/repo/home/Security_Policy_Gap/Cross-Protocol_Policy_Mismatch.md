---
title: Cross-Protocol_Policy_Mismatch
description: 
published: true
date: 2025-07-31T15:46:53.005Z
tags: 
editor: markdown
dateCreated: 2025-07-20T11:56:11.914Z
---

# 네트워크 계층 정책 불일치 (Cross-Protocol Policy Mismatch)

## 개요

“네트워크 계층 정책 불일치”는 네트워크 경계에서 설정된 보안 규칙이 트래픽을 하나의 프로토콜 흐름으로 가정하지만, 실제 통신은 암호화・캡슐화・세션 재활용 등으로 그 가정을 무너뜨려 정책 판단과 실동작 사이에 불일치를 유발한다. 공격자는 이 간극을 이용해 서로 다른 프로토콜 메시지를 숨기거나 혼합해 전송하며, 경계 장비가 오인한 통로를 통해 내부 서비스 접근, 세션 탈취, 명령 주입 같은 심각한 보안 영향을 끌어낼 수 있다.

## 분류

보안 정책 적용 불일치 (Security Policy Gap) - 네트워크 계층 정책 불일치 (Cross-Protocol Policy Mismatch)

## 하위 항목

* [ODoH Relay SSRF](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Cross-Protocol_Policy_Mismatch/ODoH_Relay_SSRF)
* [TLS Record Fragmentation](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Cross-Protocol_Policy_Mismatch/TLS_Record_Fragmentation)
* [TLS Resumption Smuggling](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Cross-Protocol_Policy_Mismatch/TLS_Resumption_Smuggling)

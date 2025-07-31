---
title: Client-Side Security Policy Mismatch
description: 
published: true
date: 2025-07-31T15:42:46.381Z
tags: 
editor: markdown
dateCreated: 2025-07-23T11:03:32.812Z
---

# 클라이언트 보안 정책 불일치 (Client-Side Security Policy Mismatch)

## 개요

“클라이언트 보안 정책 불일치”는 브라우저가 적용하는 동일 출처 정책(SOP)・콘텐츠 보안 정책(CSP)・Cookie SameSite・MIME・CDN 분리 규칙 등을 서버・프록시・콘텐츠 경로가 다른 의미로 상호 작용하면서, 클라이언트의 정책이 회피되는 틈을 노린다. 공격자는 이 간극을 활용해 보안 정책의 실제 보호 범위 밖으로 요청・코드・쿠키를 유도한다. 그 결과 XSS・세션 탈취・권한 상승・대규모 캐시 오염과 같은 사용자 중심 보안 위험이 발생한다.

## 분류

보안 정책 적용 불일치 (Security Policy Gap) - 클라이언트 보안 정책 불일치 (Client-Side Security Policy Mismatch)

## 하위 항목

* [Same-Site Cross-Origin Request Forgery](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Client-Side_Security_Policy_Mismatch/Same-Site_Cross-Origin_Request_Forgery)
* [OAuth Flow Confusion Attack](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Client-Side_Security_Policy_Mismatch/OAuth_Flow_Confusion_Attack)
* [MIME Sniffing CSP Bypass](https://semanticgap.mjsec.kr/en/home/Security_Policy_Gap/Client-Side_Security_Policy_Mismatch/MIME_Sniffing_CSP_Bypass)

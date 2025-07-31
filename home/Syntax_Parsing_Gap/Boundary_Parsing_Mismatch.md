---
title: Boundary_Parsing_Mismatch
description: 
published: true
date: 2025-07-31T15:57:56.735Z
tags: 
editor: markdown
dateCreated: 2025-07-20T11:29:22.829Z
---

# 데이터 경계 파싱 불일치 (Boundary Parsing Mismatch)

## 개요

“데이터 경계 파싱 불일치”는 하나의 프로토콜・포멧에서 요소 간 경계를 구분하는 문법(CRLF, Content-Length, 멀티파트 바운더리, XML 태그 등)을 두 개 이상의 파서가 서로 다른 규칙으로 해석할 때 발생한다. 공격자는 같은 바이트 흐름에 경계 문자를 교묘히 배치해 검증・중계 계층이 보기에 정상적인 단일 메시지처럼 보이도록 만든 뒤, 실행 파서가 이를 두 개 이상의 메시지(또는 추가 바디)로 분리해 처리하게 하여 요청 삽입, 명령 주입, 캐시 오염, 인증 우회를 일으킬 수 있다.

## 분류

구문 분석 불일치 (Syntax Parsing Gap) - 데이터 경계 파싱 불일치 (Boundary Parsing Mismatch)

## 하위 항목

* [XMPP Stanza Smuggling](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Boundary_Parsing_Mismatch/XMPP_Stanza_Smuggling)
* [Multipart Parser Confusion](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Boundary_Parsing_Mismatch/Multipart_Parser_Confusion)
* [HTTP Request Smuggling](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Boundary_Parsing_Mismatch/HTTP_Request_Smuggling)
* [Email Atom Splitting Attack](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Boundary_Parsing_Mismatch/Email_Atom_Splitting_Attack)
* [HTTP Response Queue Poisoning](https://semanticgap.mjsec.kr/en/home/Syntax_Parsing_Gap/Boundary_Parsing_Mismatch/HTTP_Response_Queue_Poisoning)

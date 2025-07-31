---
title: TLS Resumption Smuggling
description: 
published: true
date: 2025-07-31T15:00:58.257Z
tags: attack success validation, data exfiltration, server support & censorship analysis, censored content access, session ticket acquisition via tunnel, sni spoofing during resumption
editor: markdown
dateCreated: 2025-07-25T10:15:55.075Z
---

# TLS Resumption Smuggling

## 개요

“TLS Resumption Smuggling”은 TLS 핸드셰이크의 ClientHello 메시지를 다수의 fragment로 나누어 전송한 후, 해당 세션을 기반으로 TLS 세션 재개(Session Resumption)를 활용하여 검열 시스템의 탐지를 우회하는 기법이다. 초기 연결에서는 TLS 메시지 조각화(fragmentation)을 통해 SNI 필드 탐지를 실패하게 만들고, 이후의 연결에서는 기존 세션 정보를 재사용함으로써 ClientHello 자체를 생략하게 된다. 이로 인해 공격자는 TLS 연결 전체를 검열 장비의 탐지 경로 바깥에서 수행할 수 있으며, 이는 계층 간 정책 적용 방식의 불일치로 인해 발생하는 검열 우회 효과를 유도한다.

## 분류

> 보안 정책 적용 불일치 (Security Policy Gap)
> → 네트워크 계층 정책 불일치 (Cross-Protocol Policy Mismatch)
> → TLS Resumption Smuggling

## 절차

| Phase          | Description                                                                                                                                            |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Reconnaissance | 차단 대상 도메인과 TLS 1.2 세션 지원이 가능한지 알아본다. 검열 장비가 SNI, 서명서 검사로 차단하는 것을 확인하고, 외부 암호화 프록시 엔드포인트를 확보한다.                                                         |
| Tunneling      | 차단되지 않는 프록시를 통해 암호화된 TCP 터널을 열고, 그 안에서 목표 서버랑 정상적인 TLS 핸드셰이크를 수행해서 세션 티켓을 얻는다. 이 때, SNI인증서는 ISP에게 보이지 않는다.                                             |
| Evasion        | ISP가 보는 평문 경로로 서버 IP에 직접 접속하여 ClineHello에는 가짜 SNI랑 Tunneling으로 얻은 세션 티켓을 포함한다. 이 때, 많은 서버가 SNI 일치를 다시 검증하지 않기 때문에 세션이 다시 이루어져 핸드셰이크가 “Blind” 상태로 완료된다. |
| Validation     | 짧은 HTTP 요청을 보내서 세션이 실제 재개되었는지 확인하고, 실패하면 다른 가짜 SNI를 사용하거나 서버를 교체한다.                                                                                    |
| Exploitation   | 검열을 우회한 TLS 세션으로 차단 콘텐츠를 지속적으로 전송하고 수신한다.                                                                                                              |
| Exfiltration   | 우회된 채널을 이용해서 대용량 파일을 다운로드 및 업로드 하고 추가 데이터 수집 등을 수행한다.                                                                                                  |

<div style="text-align: left;">
  <img
    src="/tls_resumption_smuggling.png"
    alt="TLS Resumption Smuggling 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>

## 공격기법

### Reconnaissance

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Censorship &amp; Proxy Discovery</td>
      <td>Reliance Jio ISP의 차단 목록 분석, 외부 TCP 프록시</td>
      <td>공격 대상 ISP가 SNI 필터링이나 인증서 검사를 통해 특정 도메인을 차단하는지 확인한다. 동시에, 초기 핸드셰이크를 숨기는 데 사용할, ISP의 검열 범위 밖에 있는 암호화된 TCP 프록시 엔드포인트를 확보한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Tunneling

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Session Ticket Acquisition via Tunnel</td>
      <td>openssl s_client를 SSH 터널로 실행하여 blocked.com에 접속</td>
      <td>확보된 암호화 프록시를 통해 차단 대상 서버와 완전한 TLS 1.2 핸드셰이크를 수행한다. 이 과정은 검열 시스템에 노출되지 않으므로, 서버로부터 후속 공격에 사용할 유효한 세션 재개 티켓(Session Resumption Ticket)을 안전하게 획득할 수 있다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Evasion

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>SNI Spoofing during Resumption</td>
      <td>ClientHello의 SNI 필드 값을 blocked.com 대신 allowed.com으로 설정</td>
      <td>프록시 없이 서버에 직접 TLS 세션 재개를 요청하되, ClientHello 메시지에 포함된 SNI 값을 검열되지 않는 임의의 도메인으로 조작한다. 다수 서버는 유효한 세션 티켓이 존재할 경우 SNI 값을 검증하지 않으므로, 검열 시스템을 속이고 "Blind" 세션을 수립한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Validation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Session Resumption Confirmation</td>
      <td>초기 핸드셰이크와 재개된 세션에서 각각 받은 웹 페이지를 비교</td>
      <td>우회된 세션을 통해 간단한 HTTP 요청을 보내, 이전에 차단되었던 콘텐츠에 정상적으로 접근 가능한지 확인한다. 원본 문서에서는 초기 접속 시 저장한 웹페이지와 재개된 세션에서 받은 웹페이지가 동일한지 비교하여 공격 성공을 검증했다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exploitation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Censored Content Access</td>
      <td>-</td>
      <td>성공적으로 수립된 “blind" TLS 세션을 통해, 이전에는 접근이 불가능했던 검열된 웹사이트나 온라인 서비스에 접속하여 콘텐츠를 지속적으로 요청하고 상호작용한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exfiltration

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Secure Data Transfer</td>
      <td>-</td>
      <td>검열을 완전히 회피하여 수립된 암호화 채널을 통해, 차단되었던 콘텐츠를 다운로드하거나 민감한 데이터를 외부 서버로 안전하게 전송하는 등 정보 유출을 완료한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [https://sambhav.info/files/blindtls-foci21.pdf](https://sambhav.info/files/blindtls-foci21.pdf)

## 기타 참고문헌

\[A] [https://datatracker.ietf.org/doc/html/rfc5077](https://datatracker.ietf.org/doc/html/rfc5077)

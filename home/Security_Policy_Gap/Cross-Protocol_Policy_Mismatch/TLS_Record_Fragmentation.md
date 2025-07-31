---
title: TLS Record Fragmentation
description: 
published: true
date: 2025-07-30T19:46:55.233Z
tags: attack success validation, data exfiltration, server support & censorship analysis, clienthello fragmentation, dpi reassembly failure, censored content access
editor: markdown
dateCreated: 2025-07-25T10:15:25.426Z
---

# TLS Record Fragmentation

## 개요

“TLS Record Fragmentation”은 TLS 핸드셰이크 메시지를 여러 개의 작은 레코드로 나누어 전송함으로써, 중간의 네트워크 보안 장비(DPI, 방화벽 등)가 메시지 전체를 조립하지 못하게 만드는 방식의 기법이다. TLS 명세에서는 이러한 조각화된 전송을 정상 동작으로 간주하지만, 일부 검열 시스템이나 보안 장비는 완전한 메시지 단위로만 정책을 적용하기 때문에, 분할된 레코드 기반의 요청은 필터링을 우회할 수 있다. 이처럼 서로 다른 계층 간의 불일치로 인해 검열 우회 및 차단 실패가 발생할 수 있다.

## 분류

> 보안 정책 적용 불일치 (Security Policy Gap)
> → 네트워크 계층 정책 불일치 (Cross-Protocol Policy Mismatch)
> → TLS Record Fragmentation

## 절차

| Phase          | Description                                                                                       |
| -------------- | ------------------------------------------------------------------------------------------------- |
| Reconnaissance | 검열 시스템이나 보안 장비가 SNI 필드를 기반으로 TLS 연결을 차단함을 확인하고, TLS record fragmentation를 지원하는 클라이언트 서버 비율을 파악한다. |
| Fragmentation  | ClientHello와 같은 메시지를 여러 TLS 레코드로 분할해서 SNI 확장을 서로 다른 레코드에 나눠 담는다.                                  |
| Validation     | 소량의 트래픽(curl 등)으로 연결이 차단되지 않고 진행되는지 확인한다. 실패 시 Early/Late split 나 TCP 세그먼트 조합을 재조정한다.             |
| Evasion        | 검열 시스템이나 보안 장비가 단편화된 레코드로 인해 SNI를 해석하지 못하고 연결을 허용함으로 인해 TLS Record Fragmentation에 대한 우회가 가능해진다.   |
| Exploitation   | 검열을 우회한 TLS 세션으로 차단된 사이트에 접속하고 콘텐츠를 요청한다.                                                         |
| Exfiltration   | 우회된 채널을 통해 콘텐츠 다운로드, 데이터 전송 등을 수행한다.                                                              |

<div style="text-align: left;">
  <img
    src="/tls_record_fragmentation.png"
    alt="TLS Record Fragmentation 다이어그램"
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
      <th>공격 기법</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Server Support & Censorship Analysis</td>
      <td>Tranco Top 1M 및 CitizenLab 목록 스캔</td>
      <td>인터넷의 주요 웹사이트(Tranco Top 1M) 및 알려진 검열 대상 도메인 목록(CitizenLab)을 스캔하여, 90% 이상의 서버가 TLS 레코드 조각화를 지원함을 확인한다. 이를 통해 공격의 성공 가능성을 정찰하고, 검열 시스템이 SNI 필드를 기반으로 TLS 연결을 차단하는지 판별한다.</td>
      <td>[1],</td>
    </tr>
  </tbody>
</table>

### Fragmentation

<table>
  <thead>
    <tr>
      <th>공격 기법</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ClientHello Fragmentation</td>
      <td>DPYProxy를 이용한 핸드셰이크 메시지 분할</td>
      <td>ClientHello 메시지를 여러 개의 작은 TLS 레코드로 분할한다. 특히 검열의 주요 대상인 SNI(Server Name Indication) 확장 필드가 여러 레코드에 걸쳐 나뉘도록 조작하여, 보안 장비가 단일 레코드에서 전체 SNI를 식별하지 못하도록 만든다.</td>
      <td>[1], [2]</td>
    </tr>
  </tbody>
</table>

### Validation

<table>
  <thead>
    <tr>
      <th>공격 기법</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Circumvention Confirmation</td>
      <td><pre><code>curl -Ls --proxy 127.0.0.1:4433 https://wikipedia.org/</code></pre></td>
      <td>조각화 프록시를 통해 검열된 도메인(예: wikipedia.org)으로 curl과 같은 도구를 이용해 테스트 요청을 보낸다. 직접 연결 시 차단되던 요청이 성공적으로 응답을 받아오는지 확인하여, 우회 공격이 유효한지 검증한다.</td>
      <td>[1], [2]</td>
    </tr>
  </tbody>
</table>

### Evasion

<table>
  <thead>
    <tr>
      <th>공격 기법</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>DPI Reassembly Failure</td>
      <td>SNI가 [SN]과 [I]로 분할된 레코드</td>
      <td>검열 시스템의 DPI(Deep Packet Inspection)가 분할된 TLS 레코드를 성공적으로 재조립하지 못한다. 결과적으로 전체 SNI 값을 추출할 수 없게 되어 필터링 규칙을 적용하지 못하고, 해당 연결을 정상적인 트래픽으로 오인하여 통과시킨다.</td>
      <td>[1], [2]</td>
    </tr>
  </tbody>
</table>

### Exploitation

<table>
  <thead>
    <tr>
      <th>공격 기법</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Censored Content Access</td>
      <td>-</td>
      <td>성공적으로 우회된 TLS 세션을 통해, 이전에는 접근이 불가능했던 검열된 웹사이트나 온라인 서비스에 접속하여 콘텐츠를 자유롭게 요청하고 상호작용한다.</td>
      <td>[1], [2]</td>
    </tr>
  </tbody>
</table>

### Exfiltration

<table>
  <thead>
    <tr>
      <th>공격 기법</th>
      <th>예시</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Secure Data Transfer</td>
      <td>-</td>
      <td>검열을 회피하여 수립된 암호화된 채널을 통해, 차단되었던 콘텐츠를 다운로드하거나 민감한 데이터를 외부 서버로 안전하게 전송하는 등 정보 유출을 완료한다.</td>
      <td>[1], [2]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [Circumventing the GFW with TLS Record Fragmentation | System Security Group](https://upb-syssec.github.io/blog/2023/record-fragmentation/#tcp-fragmentation)
* \[2] [https://censorbib.nymity.ch/pdf/Niere2023a.pdf](https://censorbib.nymity.ch/pdf/Niere2023a.pdf)

## 기타 참고문헌

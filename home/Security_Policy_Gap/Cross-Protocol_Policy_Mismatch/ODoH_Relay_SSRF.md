---
title: ODoH Relay SSRF
description: 
published: true
date: 2025-07-31T18:51:53.592Z
tags: odoh endpoint discovery, request obfuscation, 307 post‑preserve, arbitrary content‑type, https → http redirect, ssrf payload injection, dns/metadata probe, internal host injection, blind port scan, https‑to‑http redirect bypass, aws imds dump, internal dashboard access, non‑standard port, relay response harvest, json credential dump
editor: markdown
dateCreated: 2025-07-23T11:06:55.707Z
---

# ODoH Relay SSRF

## 개요

“ODoH Relay SSRF”는 ODoH(Oblivious DNS over HTTPS) 프로토콜을 구현한 프록시 서버에서, 요청 시 목적지 도메인을 적절히 검증하지 않아 발생하는 문제다. Cloudflare의 odoh-server-go 를 비롯한 일부 ODoH 프록시 구현체가 DNS 요청의 목적지 유효성을 충분히 검사하지 않아, 프록시가 악의적으로 선택된 임의의 서버와 통신할 수 있다. 공격자는 이를 이용해 ODoH 프록시를 의도하지 않은 타겟 서버에 대한 HTTP 요청 전달자로 악용할 수 있으며, 결과적으로 SSRF 및 우회 접근 등 의도치 않은 네트워크 접근 권한을 얻을 수 있다.

## 분류

> 보안 정책 적용 불일치 (Security Policy Gap)  
> → 네트워크 계층 정책 불일치 (Cross-Protocol Policy Mismatch)  
> → ODoH Relay SSRF

## 절차

| Phase         | Description                                                                                     |
|---------------|-------------------------------------------------------------------------------------------------|
| Reconnaissance| ODoH Relay 엔드포인트를 확인하고 (`/.well-known/odohconfigs`, DNS records), `?targethost=` 템플릿을 조사한다. |
| Obfuscation   | WAF/필터링 우회를 위해 쿼리, 헤더 값을 Percent-encoding, 라인 폴딩 등을 이용해 다양하게 변형하여 WAF나 필터링 로직이 공격 페이로드를 탐지하지 못하도록 한다. |
| Injection     | 악성 도메인이나 내부 IP를 `targethost` 에, 원하는 경로를 `targetpath` 에 삽입하여 ODoH-relay에 요청 전송한다. |
| Validation    | SSRF-canary(공격자 제어 도메인)에 대한 DNS/HTTP 요청이 실제로 발생하는지 확인한다. |
| Bypass        | ODoH-Relay 가 HTTPS -> HTTP 리다이렉트를 검증없이 수행하는 점을 이용하여, 내부 HTTP 서비스로 우회를 요청한다. |
| Exploitation  | 우회된 경로를 통해 AWS IMDS, 내부 관리 API, 관리자 패널 등에 접근하여 자격증명·민감 정보를 수집한다. |
| Exfiltration  | 내부 서비스가 반환하는 응답을 암호화된 ODoH 메시지로 전달받아 복호화 후 민감 정보를 탈취한다. |

<div style="text-align: left;">
  <img
    src="/odoh_relay_ssrf.png"
    alt="ODoH Relay SSRF 다이어그램"
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
      <td>ODoH Endpoint Discovery</td>
      <td><pre><code>GET /.well-known/odohconfigs</code></pre></td>
      <td>ODoH Relay의 엔드포인트를 식별하고, SSRF에 이용할 수 있는 URI 템플릿 구조와 지원 host 목록을 파악한다. 이를 통해 공격 설계가 가능해진다.</td>
      <td>[1], [2], [3]</td>
    </tr>
  </tbody>
</table>

### Obfuscation
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
      <td>Request Obfuscation</td>
      <td><pre><code>targetpath=%2f..%2fapi%2f</code></pre></td>
      <td>Percent-encoding 등으로 targetpath에 우회 구문을 적용해, 필터링 로직이나 WAF, 단순 패턴 차단을 우회하려 시도한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>307 POST-Preserve</td>
      <td>307 Redirect</td>
      <td>307 리디렉션으로 HTTP 바디를 유지하며 Relay가 실제 대상에 동일 페이로드를 전달하게 만들어 요청 구조를 복잡하게 우회시킨다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Injection
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
      <td>Arbitrary Content-Type</td>
      <td><pre><code>Content-Type: application/json</code></pre></td>
      <td>ODoH는 application/oblivious-dns-message 타입을 요구하지만, Relay가 Content-Type을 제한·검증하지 않으면 공격자가 JSON 등 임의 데이터를 삽입해 Target의 비동기 API 등에 SSRF를 유발할 수 있다.</td>
      <td>[1], [2]</td>
    </tr>
    <tr>
      <td>HTTPS → HTTP Redirect</td>
      <td><pre><code>Redirect HTTPS → http://169.254.169.2 54/</code></pre></td>
      <td>Relay가 프로토콜을 엄격히 검증하지 않고, HTTPS → HTTP 리디렉션을 허용한다면 Target이 내부 HTTP 서비스로 리디렉션된 요청을 그대로 수용할 수 있다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>SSRF Payload Injection</td>
      <td><pre><code>targethost=127.0.0.1&targetpath=/admin
targethost=169.254.169.254&targetpath=/latest/meta-data/</code></pre></td>
      <td>Relay는 targethost/targetpath와 같이 민감한 내부 주소나 메타데이터 서비스 등으로의 요청도 별도 검증 없이 Target에 프록시한다. Relay는 쿼리를 "이해"하지 않고 패킷만 단순 중계하며, Target만이 실질적으로 이를 내부 리소스에 접근·처리한다.</td>
      <td>[1], [2], [3]</td>
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
      <td>DNS/Metadata Probe</td>
      <td><pre><code>GET /.well-known/odohconfigs
targethost=example.com</code></pre></td>
      <td>ODoH 구성이나 자신의 서버 등 제어 가능한 대상에 요청 후, Relay의 Proxy-Status, 응답코드, 에러 메시지, Target의 실제 응답값 등 검증 방식을 단계별로 분석한다.</td>
      <td>[1], [2], [3]</td>
    </tr>
    <tr>
      <td>Internal Host Injection</td>
      <td><pre><code>targethost=10.0.0.5</code></pre></td>
      <td>타겟 주소로 내부망·사설IP 등을 넣어, 요청이 실제 Target까지 도달하는지 상태 코드·응답 등 차이를 관찰한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Blind Port Scan</td>
      <td><pre><code>targethost:port</code></pre></td>
      <td>다양한 포트로 요청 시도 후, 타임아웃/상태 코드/응답 유무 등 타겟 서비스 유무를 추론한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Bypass
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
      <td>HTTPS-to-HTTP Redirect Bypass</td>
      <td><pre><code>targethost=attacker.com&targetpath=/redir</code></pre></td>
      <td>Relay의 맹목적 리디렉션 추적을 악용한다. 공격자 HTTPS 서버에서 내부망 HTTP 서비스로 301/302 리디렉션하여 프로토콜 제한을 우회하고, 내부망에 접근한다.</td>
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
      <td>AWS IMDS Dump</td>
      <td><pre><code>Redirect → http://169.254 .169.254/lates t/meta-data/</code></pre></td>
      <td>ODoH Relay가 HTTP 리디렉션을 맹목적으로 추적하는 점을 악용하여, 공격자 서버를 경유해 내부 클라우드 인스턴스의 메타데이터(IMDS)에 접근하고 임시 자격증명을 유출한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Internal Dashboard Access</td>
      <td><pre><code>targethost=int .example :9090</code></pre></td>
      <td>targethost 파라미터에 내부망 호스트와 특정 포트를 직접 지정하여, ODoH Relay의 필터링 부재를 이용해 Grafana 등 비공개 대시보드에 접근하고 내부 시스템 정보를 노출시킨다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Non-standard Port</td>
      <td><pre><code>targetpath=/status:8080</code></pre></td>
      <td>포트 필터링이 미흡하면 Relay가 내부 서비스·관리 포트 등에도 요청을 넘겨주므로, Target에서 민감 자원 접근 및 데이터 유출이 가능하다.</td>
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
      <td>Relay Response Harvest</td>
      <td>반복적으로 캐시 · 로그로 Relay 응답 수집</td>
      <td>자동화된 클라이언트로 SSRF 공격을 반복 실행하고 암호화된 Relay 응답을 지속적으로 수집 및 복호화하여, 내부 시스템에서 유출된 민감 정보를 지속적으로 확보한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>JSON Credential Dump</td>
      <td>IMDS JSON 그대로 공격 서버 저장</td>
      <td>EC2 인스턴스의 임시 Access Key, Secret Key 등이 담긴 JSON 데이터를 확보하고, 이를 통해 클라우드 리소스에 대한 접근 권한을 획득한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

[1] https://github.com/cloudflarearchive/odoh-server-go/issues/30  
[2] https://www.rfc-editor.org/rfc/rfc9230  
[3] https://blog.cloudflare.com/oblivious-dns/

## 기타 참고문헌


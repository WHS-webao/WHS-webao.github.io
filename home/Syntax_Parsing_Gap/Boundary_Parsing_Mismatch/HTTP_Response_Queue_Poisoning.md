---
title: HTTP Response Queue Poisoning
description: 
published: true
date: 2025-07-30T20:02:33.414Z
tags: attack success validation, data exfiltration, session hijacking, header injection discovery, connection hijacking preparation, prefixed request injection, response queue desynchronization
editor: markdown
dateCreated: 2025-07-25T09:59:04.349Z
---

# HTTP Response Queue Poisoning

## 개요

“HTTP Response Queue Poisoning”은 HTTP 응답 처리 과정에서 프록시와 백엔드 서버가 응답 경계(Content-Length, Transfer-Encoding 등)를 다르게 해석하는 점을 악용한다. 공격자는 하나의 응답 메시지 내에 다수의 응답이 있는 것처럼 구성해, 일부 경계 정보를 무시하거나 오해하도록 만들어 후속 사용자에게 조작된 응답을 전달시킬 수 있다. 이는 WAF나 리버스 프록시 등과 백엔드 서버 간의 응답 경계 처리 방식 불일치로 인해 발생하며, 최종적으로 사용자 세션 탈취, 인증 우회 등의 공격으로 이어질 수 있다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 데이터 경계 파싱 불일치 (Boundary Parsing Mismatch)
> → HTTP Response Queue Poisoning

## 절차

| Phase          | Description                                                                                                                                                     |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 대상 애플리케이션에 CRLF 기반 헤더 인젝션이 존재하는지 파악한다. 프론트엔드, 백엔드 연결이 keep-alive 상태로 재사용되고, 동일 TCP 연결로 연속 요청을 허용하는지 테스트한다.                                                      |
| Mutation       | 페이로드에 CRLF를 삽입해서 헤더 컨텍스트를 종료한 뒤, Host:, Connection: keep-alive 등을 주입하여 연결을 유지하도록 만든다. 또한, 공백, URL 인코딩 등으로 필터를 우회해서 서버가 정상 요청처럼 처리하도록 한다.                        |
| Injection      | 조작된 첫번째 요청을 전송하여 백엔드가 의도치 않은 두번째 요청의 프리픽스를 처리하도록 유도한다. 예: `GET /%20 HTTP/1.1<CRLF>Host: redacted.net<CRLF>Connection: keep-alive<CRLF><CRLF>` 뒤에 추가 라인을 이어 붙인다. |
| Poisoning      | 백엔드가 두 개의 응답을 생성하지만 프론트엔드는 하나만 예상한다. 두번째 응답이 큐에 남아 다음 사용자의 요청과 매칭되어 응답과 요청이 어긋난다.                                                                               |
| Validation     | 공격자가 같은 TCP 연결을 계속 사용하거나 반복적인 요청을 발송하여 다른 사용자의 응답이 자신에게 섞여 도착하는지 확인한다.                                                                                          |
| Exploitation   | 큐에 남긴 2차 요청/응답을 활용해 다른 사용자의 인증된 콘텐츠나 토큰을 탈취하거나 로그인 흐름을 가로채거나, 세션 하이재킹 등을 수행한다.                                                                                  |
| Exfiltration   | 획득한 자격 및 데이터를 외부 서버로 전송하거나 추가 내부 시스템 침투에 사용한다.                                                                                                                  |

<div style="text-align: left;">
  <img
    src="/http_response_queue_poisoning.png"
    alt="HTTP Response Queue Poisoning 다이어그램"
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
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Header Injection Discovery</td>
      <td>
        <pre><code>GET /%20HTTP/1.1…</code></pre>
      </td>
      <td>CRLF 문자를 URL 경로에 주입하여 프론트엔드 서버에서 헤더 인젝션이 가능한지, 프론트엔드와 백엔드 간 연결이 keep-alive로 재사용되는지 확인하여 공격 가능성을 정찰한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Mutation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Connection Hijacking Preparation</td>
      <td>
        <pre><code>…%0d%0aHost:%20redacted.net%0d%0aConnection:%20keep-alive%0d%0a%0d%0a</code></pre>
      </td>
      <td>CRLF(%0d%0a)를 이용해 현재 요청의 컨텍스트를 종료하고, Host 헤더와 Connection: keep-alive 헤더를 삽입하여 백엔드 서버가 응답 후에도 연결을 유지하도록 페이로드를 변조한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Injection

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Prefixed Request Injection</td>
      <td>
        <pre><code>GET /%20HTTP/1.1%0d%0aHost:%20redacted.net%0d%0aConnection:%20keep-alive%0d%0a%0d%0aGET%20/%20HTTP/1.1%0d%0aFoo:%20bar HTTP/1.1</code></pre>
      </td>
      <td>연결이 유지된 상태에서 다음 사용자의 요청과 결합될 악의적인 요청 프리픽스를 주입한다. 이 프리픽스는 서버가 추가하는 나머지 헤더/바디와 결합하여 완전한 두 번째 요청을 형성하도록 설계된다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Poisoning

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Response Queue Desynchronization</td>
      <td>
        <pre><code>(Backend sends two responses for one request)</code></pre>
      </td>
      <td>프론트엔드는 단일 요청을 보냈다고 인식하지만, 백엔드는 주입된 프리픽스로 인해 두 개의 요청을 수신하고 두 개의 응답을 생성한다. 프론트엔드가 첫 번째 응답만 소비하면서, 두 번째 응답이 큐에 남아 다음 요청과 어긋나게 매칭된다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Validation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Intermittent Response Interception</td>
      <td>-</td>
      <td>응답 큐가 오염된 후, 공격자가 후속 요청을 보내 다른 사용자의 응답을 간헐적으로 수신하는지 확인함으로써 공격 성공 여부를 검증한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exploitation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Cross-User Session Hijacking</td>
      <td>-</td>
      <td>큐에 남겨진 응답을 가로채 다른 사용자의 민감한 데이터, 세션 쿠키, 인증 토큰 등을 탈취하여 해당 사용자의 세션을 하이재킹한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exfiltration

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Privileged Data Exfiltration</td>
      <td>-</td>
      <td>성공적으로 탈취한 세션이나 데이터를 사용하여 보호된 리소스에 접근하고, 공격자에게 피해자 계정에 대한 완전한 접근 권한을 부여한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [https://portswigger.net/research/making-http-header-injection-critical-via-response-queue-poisoning](https://portswigger.net/research/making-http-header-injection-critical-via-response-queue-poisoning)

## 기타 참고문헌

* \[A] [https://portswigger.net/research/http2#splitting](https://portswigger.net/research/http2#splitting)
* \[B] [https://portswigger.net/web-security/request-smuggling/advanced#request-smuggling-via-crlf-injection](https://portswigger.net/web-security/request-smuggling/advanced#request-smuggling-via-crlf-injection)

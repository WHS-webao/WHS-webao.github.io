---
title: Apache Filename Confusion
description: 
published: true
date: 2025-07-30T20:38:29.118Z
tags: attack success validation, authorization bypass, configuration analysis, module interpretation differential, path delimiter encoding, truncation payload injection, path traversal, arbitrary file read
editor: markdown
dateCreated: 2025-07-25T09:59:45.582Z
---

# Apache Filename Confusion

## 개요

“Apache Filename Confusion”은 요청 URI에 filename처럼 보이는 경로를 삽입하거나 인코딩하여, 웹 서버 내부 모듈이 이를 물리적 파일 경로로 해석할지, URL로 해석할지를 혼동하도록 유도하는 기법이다. Apache HTTP Server 등에서 r->filename 필드가 일부 모듈에는 파일 시스템 경로로, 다른 모듈에는 요청 URI나 쿼리 파라미터로 재해석되며, 이로 인해 접근 제어, 리라이트 규칙, 인증 로직 등이 우회될 수 있다. 경로를 구성하는 문자열이 파서마다 다르게 처리되면서, 동일한 요청이 보안 정책을 우회하거나 비정상적인 자원에 접근하게 된다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 경로 파싱 불일치 (Path Parsing Mismatch)
> → Apache Filename Confusion

## 절차

| Phase          | Description                                                                                             |
| -------------- | ------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 대상 서버의 모듈 조합 (특히 mod\_rewrite, mod\_proxy\_xxx, mod\_authz\_core) 및 RewriteRule, ProxyPass 등의 설정을 수집한다. |
| Divergence     | Apache 내부에서 어떤 모듈은 r->filename을 파일 시스템 경로로, 다른 모듈은 URL로 해석한다는 사실을 이용하여 파서 간 인식 차이 유발 지점을 확인한다.          |
| Mutation       | URL 인코딩으로 `/`, `?`를 각각 `%2F`, `%3F`로 변형해 모듈마다 다른 문자열로 보이게 만든다.                                          |
| Injection      | 변형된 경로(예: `/admin.php%3Fooo.php`)를 실제로 전송해 Path Truncation 또는 Authentication Bypass 프리미티브를 발동시킨다.       |
| Validation     | 동일 자원에 정상 요청과 변조된 요청을 반복해서 상태코드 차이(401↔200), 응답 본문 차이를 비교한다.                                            |
| Exploitation   | ACL/인증 우회, 민감 파일 열람, 백엔드 프록시 경유 RCE 등을 수행한다.                                                            |
| Exfiltration   | 노출된 설정 파일, 크리덴셜, 소스코드 등을 추출한다.                                                                          |

<div style="text-align: left;">
  <img
    src="/apache_filename_confusion.png"
    alt="Apache Filename Confusion 다이어그램"
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
      <td>Configuration Analysis</td>
      <td>-</td>
      <td>대상 서버의 Apache 설정 파일을 분석하여, mod_rewrite, mod_proxy_fcgi 등 모듈의 조합과, &lt;Files&gt;, RewriteRule과 같이 경로 기반으로 설정된 접근 제어 규칙을 식별한다.</td>
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
      <td>Module Interpretation Differential</td>
      <td><pre><code>r-&gt;filename</code></pre></td>
      <td>Apache 내부의 여러 모듈이 동일한 요청 구조체 필드(r-&gt;filename)를 서로 다른 의미(파일 시스템 경로 vs URL)로 해석하는 지점을 식별한다. 예를 들어 mod_authz_core는 파일 경로로, mod_proxy는 프록시 URL로 처리하여 해석 불일치를 유발한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Mutation

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
      <td>Path Delimiter Encoding</td>
      <td>
        <pre><code>/admin.php?fake.php → /admin.php%3Ffake.php</code></pre>
      </td>
      <td>경로 구분자로 사용될 수 있는 물음표(?)를 URL 인코딩하여 `%3F`로 변형한다. 이를 통해 일부 모듈은 이를 일반 문자열로 인식하게 하고, 다른 모듈이나 백엔드에서는 원래의 의미로 해석하도록 만들어 혼란을 발생시킨다.</td>
      <td>[1], [2]</td>
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
      <td>Truncation Payload Injection</td>
      <td><pre><code>admin.php%3Ffake.php</code></pre></td>
      <td>인코딩된 문자가 포함된 변형된 경로를 서버에 전송한다. 백엔드(PHP-FPM 등)가 `%3F`를 `?`로 디코딩한 후 경로를 절단하게 만들어, `admin.php`가 실행되도록 하는 인증 우회 프리미티브를 발동시킨다.</td>
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
      <td>Status Code Differential Analysis</td>
      <td>
        <pre><code>/admin.php → 401 Unauthorized  
/admin.php%3Ffake.php → 200 OK</code></pre>
      </td>
      <td>보호된 리소스에 정상 요청을 보냈을 때(401)와 변조된 요청을 보냈을 때(200)의 HTTP 상태 코드를 비교한다. 응답 코드의 차이를 통해 접근 제어 우회 성공 여부를 검증한다.</td>
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
      <td>ACL Bypass</td>
      <td><pre><code>http://server/admin.php%3fooo.php</code></pre></td>
      <td>&lt;Files&gt; 지시어로 보호되는 관리자 페이지, 설정 정보 페이지(phpinfo), 기타 민감 스크립트에 대해 파일명 혼란을 유발하여 IP 기반 접근 제한이나 기본 인증(Basic Auth)을 우회한다.</td>
      <td>[1], [2]</td>
    </tr>
    <tr>
      <td>Path Traversal</td>
      <td><pre><code>http://server/user/orange%2F..%2Fsecret.yml%3F</code></pre></td>
      <td>RewriteRule이 적용된 경로에 인코딩된 슬래시(%2F)와 물음표(%3F)를 주입한다. 이를 통해 경로를 조작하여 지정된 디렉터리를 벗어나고, 규칙에 의해 강제로 추가되는 접미사(.yml)를 무력화하여 임의의 파일을 읽는다.</td>
      <td>[1], [2]</td>
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
      <td>Arbitrary File Read</td>
      <td>
        <pre><code>/usr/share/javascript/jquery-jfeed/proxy.php%3Furl=/etc/passwd&amp;x  
→ /etc/passwd</code></pre>
      </td>
      <td>성공적인 경로 탐색(Path Traversal) 공격을 통해, 웹 루트 외부나 접근이 제한된 위치에 있는 설정 파일, 소스 코드, 비밀번호 파일 등의 내용을 읽어 HTTP 응답으로 유출한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [https://i.blackhat.com/BH-US-24/Presentations/US24-Orange-Confusion-Attacks-Exploiting-Hidden-Semantic-Thursday.pdf](https://i.blackhat.com/BH-US-24/Presentations/US24-Orange-Confusion-Attacks-Exploiting-Hidden-Semantic-Thursday.pdf)
* \[2] [https://httpd.apache.org/security/vulnerabilities\_24.html](https://httpd.apache.org/security/vulnerabilities_24.html)

## 기타 참고문헌

* \[A] [https://blog.orange.tw/](https://blog.orange.tw/)
* \[B] [https://httpd.apache.org/docs/2.4/mod/mod\_rewrite.html](https://httpd.apache.org/docs/2.4/mod/mod_rewrite.html)
* \[C] [https://httpd.apache.org/docs/2.4/mod/mod\_proxy\_fcgi.htmlhttps://httpd.apache.org/docs/2.4/mod/mod\_proxy\_fcgi.html](https://httpd.apache.org/docs/2.4/mod/mod_proxy_fcgi.htmlhttps://httpd.apache.org/docs/2.4/mod/mod_proxy_fcgi.html)

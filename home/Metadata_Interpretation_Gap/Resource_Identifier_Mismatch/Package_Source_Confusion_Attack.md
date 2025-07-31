---
title: Package Source Confusion Attack
description: 
published: true
date: 2025-07-30T18:01:09.920Z
tags: internal package name discovery, public registry poisoning, data exfiltration via dns tunneling, remote code execution via package scripts
editor: markdown
dateCreated: 2025-07-25T10:20:50.459Z
---

# Package Source Confusion Attack

## 개요

“Package Source Confusion Attack”은 패키지 매니저가 동일한 이름의 패키지를 사설 저장소와 공개 저장소 중 어디에서 우선 조회할지를 결정하는 방식의 차이를 악용한다. 공격자는 내부 개발 환경에서 사용하는 패키지 이름과 동일한 이름을 가진 악성 패키지를 공개 저장소에 먼저 등록하고, 자동화된 빌드 시스템이나 CI 환경이 이를 우선적으로 설치하도록 유도한다. 이는 저장소 해석 우선순위에 대한 불일치 또는 명시적 설정 부재로 인해 발생하며, 결과적으로 내부 네트워크에서 외부 코드 실행이나 공급망 오염이 일어날 수 있다.

## 분류

> 메타데이터 해석 불일치 (Metadata Interpretation Gap)
> → 리소스 식별자 해석 불일치 (Resource Identifier Mismatch)
> → Package Source Confusion Attack

## 절차

| Phase          | Description                                                                             |
| -------------- | --------------------------------------------------------------------------------------- |
| Reconnaissance | 공개 Github 코드, 빌드 산출물 등을 대량 스캔해서 조직의 사설 패키지 이름을 수집한다.                                    |
| Poisoning      | 수집한 이름으로 공개 레지스트리(npm, PyPI, RubyGems)에 악성 패키지를 업로드해서 공급망을 오염시킨다.                       |
| Divergence     | 패키지 관리자가 내부, 외부 소스 중 버전이 높은 쪽을 자동으로 선택하는 출처 결정 로직을 따라 불일치를 유발한다.                        |
| Evasion        | DNS 트래픽처럼 잘 차단되지 않는 채널을 이용하거나, 사설 공개 저장소가 혼합된 virtual repo 설정을 악용하여 보안 장비, 보안 정책을 회피한다. |
| Exploitation   | 빌드, CI 서버, 개발자 PC가 공격자가 올린 패키지를 설치하면서 원격 코드 실행 (RCE)가 일어난다.                             |
| Validation     | 스크립트가 보낸 DNS 콜백을 확인해 “어디서 패키지가 실행됐는지” 파악한다.                                             |
| Exfiltration   | preinstall 스크립트가 수집한 호스트, 사용자 정보를 DNS 쿼리로 외부 서버에 전송하여 네트워크를 우회, 로깅한다.                   |

<div style="text-align: left;">
  <img
    src="/package_source_confusion_attack.png"
    alt="Package Source Confusion Attack 다이어그램"
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
      <td>Internal Package Name Discovery</td>
      <td>
        <pre><code>package.json 파일 내
"private-dependency-name"</code></pre>
      </td>
      <td>package.json 파일에 포함된, 공개 레지스트리에는 존재하지 않는 private dependencies(사설 의존성)의 이름을 찾아낸다.</td>
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
      <td>Public Registry Poisoning</td>
      <td>
        <pre><code>package.json 파일 내
"preinstall": "[악성 스크립트]"</code></pre>
      </td>
      <td>npm의 preinstall 스크립트 실행 기능을 이용하여, 패키지가 설치되는 시점에 악성 코드가 자동으로 실행되도록 패키지를 제작하고 공개 저장소에 업로드한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Divergence

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
      <td>Version Hijacking</td>
      <td>
        <pre><code>library 9000.0.0</code></pre>
      </td>
      <td>내부용 패키지보다 월등히 높은 버전(예: 9000.0.0)을 가진 악성 패키지를 공개 저장소에 업로드한다. pip와 같은 패키지 매니저는 더 높은 버전의 패키지를 우선적으로 설치한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Evasion

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
      <td>DNS Exfiltration for Firewall Evasion</td>
      <td>
        <pre><code>[hex-encoded-data].your-server.com</code></pre>
      </td>
      <td>탈취할 데이터를 16진수(hex)로 인코딩한 뒤, DNS 쿼리의 서브도메인으로 만들어 외부 서버로 전송한다. 이 방식은 일반적인 네트워크 방화벽을 우회하기 용이하다.</td>
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
      <td>Remote Code Execution via Package Scripts</td>
      <td>
        <pre><code>-</code></pre>
      </td>
      <td>preinstall 스크립트가 실행되면서 공격자가 사전에 정의한 코드가 내부 시스템에서 실행된다.</td>
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
      <td>DNS Callback Verification</td>
      <td>
        <pre><code>[hex-encoded-data].your-server.com</code></pre>
      </td>
      <td>공격자가 자신의 DNS 서버 로그를 확인한다. 특정 서브도메인을 포함한 DNS 쿼리가 수신되면, 이는 악성 코드가 특정 시스템에서 성공적으로 실행되었음을 검증하는 콜백 역할을 한다.</td>
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
      <td>Data Exfiltration via DNS Tunneling</td>
      <td>
        <pre><code>hex-encoded(username + hostname + path)</code></pre>
      </td>
      <td>preinstall 스크립트를 통해 시스템의 username, hostname, current path 정보를 수집하고, 이를 16진수로 인코딩하여 DNS 쿼리를 통해 외부로 유출한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610)

## 기타 참고문헌

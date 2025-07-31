---
title: Homoglyph Username Bypass
description: 
published: true
date: 2025-07-31T15:55:51.774Z
tags: attack success validation, homoglyph‑based substitution, reserved username identification, dotless i case collision, user impersonation and phishing
editor: markdown
dateCreated: 2025-07-27T10:14:00.322Z
---

# Homoglyph Username Bypass

## 개요

“Homoglyph Username Bypass”는 계정 이름에 라틴 문자를 닮은 그리스・키릴 동형문자를 섞어 등록해, 브라우저・앱 UI에서는 기존 사용자 이름과 똑같이 보이지만 백엔드 식별자는 전혀 다른 코드포인트가 되도록 만든다. 필터나 관리자 도구가 시각적 문자열만 확인해 중복・권한・신고 처리를 결정하면 공격자는 보호된 이름을 새로 확보하거나 기존 계정으로 가장할 수 있고, 이로 인해 피싱・권한 상승 같은 후속 공격으로 계정 탈취가 가능해진다.


## 분류

> 데이터 표현 모델 불일치 (Data Representation Gap)
> → 유사문자 및 정규화 처리 문제 (Homoglyph & Normalization Mismatch)
> → Homoglyph Username Bypass

## 절차

| Phase          | Description                                                                                   |
| -------------- | --------------------------------------------------------------------------------------------- |
| Reconnaissance | 타겟 시스템의 사용자 식별 방식, 유효 사용자명, 필터링 방식을 파악한다.                                                     |
| Mutation       | 정상 사용자명과 시각적으로 유사한 homoglyph를 이용하여 사용자명을 생성한다.                                                |
| Obfuscation    | 대상 시스템 필터가 강력한 경우 단순 homoglyph뿐만 아니라 혼합 인코딩, 제어문자 등을 삽입하여 난독화를 수행한다.                          |
| Validation     | 목표한 homoglyph 사용자명이 실제 등록이 가능한지, 기존 계정과 충돌하지 않는지 확인한다. 시스템이 정규화/중복 체크가 미흡할 시 우회가 성공한다.        |
| Exploitation   | 시스템 관리자, 사용자가 로그, UI 상에서 진짜 사용자와 가짜 사용자를 혼동하도록 유도한다. 시각적으로 유사한 사용자명을 악용하여 피싱/스푸핑 등의 공격을 수행한다. |

<div style="text-align: left;">
  <img
    src="/homoglyph_username_bypass.png"
    alt="Homoglyph Username Bypass 다이어그램"
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
      <td>Reserved Username Identification</td>
      <td><code>admin, support, google, SаmsungSupport</code></td>
      <td>플랫폼에서 사칭을 방지하기 위해 사용을 제한하는 관리자, 지원팀, 브랜드 이름과 같은 예약된 사용자명을 식별한다.</td>
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
      <td>Homoglyph Substitution</td>
      <td>
        <code>admin → aԁmin (d → Cyrillic 'ԁ')<br>google → gооgle (o → Cyrillic 'о')</code>
      </td>
      <td>라틴 문자와 시각적으로는 동일해 보이지만 유니코드 코드 포인트가 다른 동형문자를 사용하여, 플랫폼의 입력 유효성 검사 필터를 우회하는 새로운 사용자명을 생성한다.</td>
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
      <td>Dotless I Case Collision</td>
      <td><code>adm\u0131n</code></td>
      <td>시스템이 대소문자를 구분하지 않기 위해 사용하는 .upper()와 같은 케이스 변환(Case Conversion) 함수의 동작 방식을 악용한다. 공격자는 유니코드에서 특별한 대소문자 변환 규칙을 가진 문자(예: 점이 없는 'ı')를 사용하여 사용자명을 생성한다. 이 사용자명은 변환 후 'ADMIN'과 같은 보호된 이름과 동일해져, 단순 비교 필터를 우회하고 권한 상승과 같은 문제를 일으킨다.</td>
      <td>[A]</td>
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
      <td>Account Registration Attempt</td>
      <td><code>동형문자로 변환된 aԁmin 또는 Oрро-Support와 같은 사용자명으로 계정 등록</code></td>
      <td>동형문자로 조작된 사용자명을 이용해 플랫폼에 실제로 계정 생성을 시도한다. 필터가 이를 감지하지 못하고 계정 등록이 완료 된다면 이를 통해 우회 성공 여부를 파악할 수 있다.</td>
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
      <td>User Impersonation and Phishing</td>
      <td><code>"포럼 토론에 참여하거나, 민감한 정보를 요구하는 답글을 달거나, 공식 지원으로 위장한 악성 링크를 유포합니다."</code></td>
      <td>생성된 동형문자 계정을 사용하여 공식 지원팀이나 신뢰할 수 있는 사용자를 사칭한다. 대부분의 사용자는 시각적 차이를 인지하지 못하므로 피싱이나 소셜 엔지니어링 공격에 쉽게 노출된다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1] [https://icecream23.medium.com/i-fooled-the-filters-homoglyph-username-bypass-vulnerability-an-overlooked-threat-in-major-dd5f8cc63ba6](https://icecream23.medium.com/i-fooled-the-filters-homoglyph-username-bypass-vulnerability-an-overlooked-threat-in-major-dd5f8cc63ba6)
\[2] [https://www.securitynewspaper.com/2025/06/16/how-tokenbreak-technique-hacks-openai-anthropic-and-gemini-ai-filters-step-by-step-tutorial/](https://www.securitynewspaper.com/2025/06/16/how-tokenbreak-technique-hacks-openai-anthropic-and-gemini-ai-filters-step-by-step-tutorial/)

## 기타 참고문헌

\[A] [https://gosecure.github.io/presentations/2020\_02\_28\_confoo/unicode\_vulnerabilities\_that\_could\_bite\_you.pdf](https://gosecure.github.io/presentations/2020_02_28_confoo/unicode_vulnerabilities_that_could_bite_you.pdf)

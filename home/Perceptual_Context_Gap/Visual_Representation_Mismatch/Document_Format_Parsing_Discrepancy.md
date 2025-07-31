---
title: Document Format Parsing Discrepancy
description: 
published: true
date: 2025-07-31T16:52:36.674Z
tags: attack success validation, data exfiltration, rendering discrepancy mapping, hybrid pdf generation, document structure manipulation, business email compromise, process chain exploitation
editor: markdown
dateCreated: 2025-07-23T10:50:57.210Z
---

# Document Format Parsing Discrepancy

## 개요

“Document Format Parsing Discrepancy”는 PDF 표준상 허용된 양식 필드(Form Field)와 위젯 주석(Widget Annotation) 값의 우선순위를 렌더링 엔진마다 다르게 해석하는 불일치를 악용한다. Chrome PDFium, Firefox PDF.js, Safari WebKit PDF, Google Drive Preview 등 다양한 환경에서 이 해석 차이를 활용할 수 있다. 공격자는 단일 PDF 문서를 환경별로 다르게 렌더링되게 설계하여, 승인 과정이나 보안 필터링을 우회하거나 사용자에게 악성 콘텐츠를 표시하도록 할 수 있다. 결과적으로 정보 유출, 인증 우회 및 사회공학 공격과 같은 보안 위협을 발생시킬 수 있다.

## 분류

> 인지 맥락 불일치 (Perceptual Context Gap)  
> → 시각적 표현 불일치 (Visual Representation Mismatch)  
> → Document Format Parsing Discrepancy

## 절차


| Phase         | Description                                                                                     |
|---------------|-------------------------------------------------------------------------------------------------|
| Reconnaissance| 대상 조직이 어떤 렌더링 엔진(Safari Preview, Chrome PDFium, Google Drive, Outlook Preview 등)을 업무 흐름에서 사용하는지 확인해 표시 값을 달리 넣을 타깃 뷰어를 선정한다. |
| Divergence    | 하나의 PDF 내부에 서로 충돌하는 표현(예: AcroForm 기본값 vs. Widget Annotation Appearance Stream)을 삽입해서 뷰어마다 동일 필드를 다르게 렌더링하도록 만든다. |
| Obfuscation   | FlateDecode 압축, XObject 분리, 암호화, JavaScript Action/XFA 레이어 삽입 등으로 충돌하는 표현들을 숨겨 정적 분석 및 시각 검토를 우회한다. |
| Validation    | 공격자가 전화·채팅·스크린샷 요청, 혹은 PDF 내부에 심어 둔 원격 이미지·링크 요청 로깅으로 “어떤 값이 실제로 표시됐는지” 확인한다. |
| Exploitation  | 금액이 잘못된 버전을 본 사용자가 그대로 결제, 승인하도록 속이거나, 양식 필드에 삽입된 Javascript 액션 또는 피싱 링크를 통해 클립, 입력을 유도해서 추가 피해를 일으킨다. |
| Exfiltration  | PDF에 삽입된 /URI Action·SubmitForm 액션 또는 원격 이미지를 이용해 쿠키·세션·사용자 입력 데이터를 공격자 서버로 전송하고, 후속 공격을 준비하거나 영구 접근 토큰을 확보한다. |

<div style="text-align: left;">
  <img
    src="/document_format_parsing_discrepancy.png"
    alt="Document Format Parsing Discrepancy 다이어그램"
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
      <td>Rendering Discrepancy Mapping</td>
      <td>
        <pre><code>// Safari에서 표시될 기본값
field.setValue("£399");
// Chrome에서 표시될 annotation 값  
appearanceContents.showText("£999");</code></pre>
      </td>
      <td>동일한 폼 필드에 서로 다른 “기본값”과 “애노테이션 값”을 설정한 뒤, Safari에서는 <pre><code>field.setValue("£399")</code></pre>가, Chrome/Drive Preview에서는 <pre><code>appearanceContents.showText("£999")</code></pre>가 표시되는 조건을 식별·매핑한다.</td>
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
      <td>Hybrid PDF Generation</td>
      <td>
        <pre><code>// Safari에서 표시될 기본값
field.setValue("£399");
// Chrome에서 표시될 annotation 값  
appearanceContents.showText("£999");</code></pre>
      </td>
      <td>Apache PDFBox 라이브러리를 사용하여 Form Field와 Widget Annotation에 서로 다른 값을 설정한 하이브리드 PDF를 생성한다.</td>
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
      <td>Document Structure Manipulation</td>
      <td><pre><code>PDAcroForm.getNeedAppearances() = true
// 설정으로 기본 appearance 우회</code></pre></td>
      <td>PDF 내부 구조를 조작하여 브라우저별로 다른 렌더링 결과를 유도하면서도 정상적인 PDF로 위장한다.</td>
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
      <td>Multi-Browser Compatibility Testing</td>
      <td>Safari, Chrome, Firefox에서 각기 다른 가격 표시 확인</td>
      <td>생성된 악성 PDF가 목표 브라우저들에서 의도한 대로 서로 다른 값을 표시하는지 검증한다.</td>
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
      <td>Business Email Compromise</td>
      <td>Chrome/Firefox에서 높은 금액으로 위장된 PDF 첨부 메일</td>
      <td>기존 거래처 명의로 위장한 청구서 PDF를 메일로 보내 정상적인 비즈니스 거래로 위장하여 승인 후 회계팀이 높은 금액을 처리하도록 유도한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Process Chain Exploitation</td>
      <td>승인→전달→처리 브라우저 환경 변화 이용</td>
      <td>승인자와 처리자 간 서로 다른 브라우저 환경을 악용하여 금액 차이를 인지하지 못하게 한다.</td>
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
      <td>Financial Impact Realization</td>
      <td>£399 승인 → £999 처리</td>
      <td>승인된 금액과 실제 처리되는 금액 간의 차이를 통해 £600의 금전적 이익을 취득한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

[1] https://portswigger.net/research/fickle-pdfs-exploiting-browser-rendering-discrepancies

## 기타 참고문헌


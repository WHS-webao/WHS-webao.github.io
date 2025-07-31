---
title: YAML Key Duplication
description: 
published: true
date: 2025-07-30T20:33:07.316Z
tags: attack success validation, backend parser profiling, duplicate key injection, security scanner evasion, privilege escalation
editor: markdown
dateCreated: 2025-07-23T10:50:23.695Z
---

# YAML Key Duplication

## 개요

“YAML Key Duplication”은 하나의 YAML 문서 내에서 동일한 키가 중복 정의되었을 때, 이를 해석하는 파서마다 처리 방식이 달라지는 취약점이다. 일부 YAML 파서는 중복 키가 있을 경우 첫 번째 값을 유지하고, 다른 구현체는 마지막 값을 덮어쓰며, 또 다른 일부는 예외를 발생시키지 않고 무시하거나 병합 처리한다. 이러한 불일치는 YAML을 기반으로 구성된 보안 정책, 사용자 역할, 접근 권한, 기능 플래그 등에서 의도하지 않은 설정 오염을 유발할 수 있다. 공격자는 이를 악용해 권한 상승, 인증 우회, 보안 정책 무력화 등 설정 기반 우회 공격을 수행할 수 있으며, 결과적으로 시스템의 제어 흐름을 예측 불가능하게 만들 수 있다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 중복 키 처리 불일치 (Duplicate Key Handling Mismatch)
> → YAML Key Duplication

## 절차

| Phase          | Description                                                                                            |
| -------------- | ------------------------------------------------------------------------------------------------------ |
| Reconnaissance | 파이프라인에 투입되는 다중 YAML 파서(예: IaC 스캐너→K8S API, Ruby 전처리→Go 백엔드 등)를 파악하고 각 파서의 중복 키 처리 규칙을 조사한다.            |
| Mutation       | `!!binary` 또는 `!binary` 태그, Base64 인코딩 키 등으로 형태를 변형한다.                                                 |
| Divergence     | 하나의 맵 안에 동일 키를 두 번 이상 넣어 파서 간 해석 차이를 유발한다.                                                             |
| Validation     | 각 단계별 로그·오류·출력 확인한다. 예를 들면, Ruby 파서는 `"lang":"second"`를 수용하고 Go 파서는 오류 반환, Python은 `{'lang':'second'}` |
| Bypass         | IaC 스캐너, CI 파이프라인, K8S 어드미션 웹훅 등 보안 품질 검사를 회피하기 위해 `!!binary`·스칼라 폴딩·공백·인라인 Alias/Merge Key 등을 사용한다.   |
| Exploitation   | 검증된 중복 키 YAML을 최종 파서에 전달한다. 앞단 Ruby가 안전한 값인지 확인하여 뒷단 Go가 위험한 설정을 받아 권한 상승, 임의 파일 작성 등이 발생한다.           |
| Exfiltration   | 얻은 권한으로 자격 증명 탈취, 민감 파일·설정 가져오기, 런타임 데이터 추출 등을 수행한다.                                                   |

<div style="text-align: left;">
  <img
    src="/yaml_key_duplication.png"
    alt="YAML Key Duplication 다이어그램"
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
      <td>Parser Differential Analysis</td>
      <td>-</td>
      <td>CI/CD 파이프라인이나 애플리케이션 스택에서 사용되는 여러 YAML 파서(예: Ruby, Go)의 동작을 분석한다. 동일한 YAML 문서를 각 파서에 입력하여, 중복 키를 어떻게 처리하는지 관찰하고 해석의 차이가 발생하는 지점을 식별한다.</td>
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
      <td>Key Type Casting</td>
      <td>
        <pre><code>lang: first
!!binary bGFuZw==: second</code></pre>
      </td>
      <td>`!!binary` 태그를 사용하여 Base64로 인코딩된 키(`bGFuZw==`)를 바이너리 타입으로 강제 변환한다. 일부 파서(Ruby, Go)는 이 키를 다시 문자열 'lang'으로 해석하여 첫 번째 키와 충돌시키지만, 다른 파서(Python)는 별개의 키로 인식하여 해석의 차이를 유발한다.</td>
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
      <td>Duplicate Key Injection</td>
      <td>
        <pre><code>lang: first
lang: second</code></pre>
      </td>
      <td>하나의 YAML 맵 안에 동일한 키(lang)를 두 번 이상 정의하여 파서 간의 해석 차이를 유발한다. 이를 통해 앞단의 Ruby 파서는 `lang: second`로, 뒷단의 Go 파서는 `lang: first`로 해석하게 만들어, 후속 단계에서 논리적 충돌이나 설정 오염을 일으킨다.</td>
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
      <td>Differential Output Confirmation</td>
      <td>
        <pre><code>Ruby: {"lang"=>"second"}
Go: {"lang":"first"}</code></pre>
      </td>
      <td>중복 키가 포함된 YAML을 각 파서로 실행한 뒤, 파싱된 결과(JSON 출력, 로그, 오류 메시지)를 비교한다. 각 파서가 서로 다른 값을 반환하는지 직접 확인함으로써, 의도한 대로 해석 불일치가 발생했는지 검증한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Bypass

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
      <td>Security Scanner Evasion</td>
      <td>
        <pre><code>image: "safe-image"
image: "malicious-image"</code></pre>
      </td>
      <td>IaC(Infrastructure as Code) 스캐너와 같은 보안 도구가 첫 번째 값(`safe-image`)을 읽고 안전하다고 판단하게 만든다. 실제 런타임 환경의 파서는 마지막 값(`malicious-image`)을 채택하도록 하여, 보안 검사를 우회하고 악성 설정을 배포한다.</td>
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
      <td>Privilege Escalation</td>
      <td>
        <pre><code>user: "guest"
user: "admin"</code></pre>
      </td>
      <td>사용자 권한을 정의하는 YAML 설정에서, 보안 검증 단계에서는 `guest`로 인식되게 하고 실제 애플리케이션에서는 `admin`으로 인식되게 하여 관리자 권한을 획득한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

* \[1] [https://0day.click/parser-diff-talk-oc25/](https://0day.click/parser-diff-talk-oc25/)

## 기타 참고문헌

* \[A] [https://0day.click/parser-diff-talk-oc25/](https://0day.click/parser-diff-talk-oc25/)
* \[B] [https://justi.cz/security/2017/11/14/couchdb-rce-npm.html](https://justi.cz/security/2017/11/14/couchdb-rce-npm.html)
* \[C] [https://yaml.org/spec/1.2.2/#id2785177](https://yaml.org/spec/1.2.2/#id2785177)
* \[D] [https://tools.ietf.org/html/rfc7230](https://tools.ietf.org/html/rfc7230)

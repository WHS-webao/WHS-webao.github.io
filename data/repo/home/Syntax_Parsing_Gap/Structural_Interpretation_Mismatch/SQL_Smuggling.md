---
title: SQL Smuggling
description: 
published: true
date: 2025-07-31T16:06:29.315Z
tags: technology‑specific parsing analysis, abuse backslash escaping, multi‑level encoding bypass, attack success validation, sql injection, map signature blacklists, trigger best‑fit translation, inject inline comments, expand whitespace tokens, switch encoding forms, build sql via char()/exec()
editor: markdown
dateCreated: 2025-07-23T10:48:21.650Z
---

# SQL Smuggling

## 개요

“SQL Smuggling”은 애플리케이션이나 WAF가 입력 문자열을 무해한 데이터로 해석하더라도, 데이터베이스의 파서가 이를 SQL 구문 구조로 재해석함으로써 발생하는 취약점이다. 공격자는 이스케이프 문자('), 문자 인코딩(%CA%BC), 유니코드 문자(U+02BC) 등을 이용해 필터링을 우회하고, DBMS가 실행 시점에 이를 쿼리 구조로 복원하도록 유도한다. 이로 인해 입력 검증이나 WAF 필터를 통과한 입력이 DB 내부에서는 악성 SQL 구문으로 처리되며, 결과적으로 SQL 인젝션 방어 체계를 우회하고 인증 우회, 권한 상승, 데이터 조작 등의 공격이 가능해진다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 구조 파싱 불일치 (Structural Parsing Mismatch)
> → SQL Smuggling

## 절차

| Phase                    | Description                                                                                                                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance           | WAF·웹 서버·DB 로그에서 특수 문자(%CA%BC, %2F\* 등)가 포함된 요청을 탐지하고, DBMS 종류·문자 인코딩·필터링 규칙을 식별한다.                                                                                                               |
| Desync                   | 애플리케이션 파서와 DB 파서가 동일하게 해석하지 않는 지점을 찾는다. 예를 들어 Content-Length vs Transfer-Encoding 등의 충돌 기법을 사용해서 두 파서 사이에 불일치 하는 처리 흐름을 만들어내는 부분을 찾는다.                                                            |
| Mutation                 | 페이로드를 플랫폼별 문법,인코딩,문자셋 차이를 이용해 변형한다. 예를 들어, MySQL에서 작은따옴표를 백슬래시(') 또는 두 개의 작은따옴표('')로 이스케이프하고, URL/Hex 인코딩을 적용해 파서 해석 차이를 극대화하며, Unicode homoglyph(U+02BC 등)가 DBMS 내부에서 작은따옴표(')로 자동 변환되는 점을 악용한다. |
| Obfuscation              | /\*\*/ 주석 삽입, 대량 공백·개행, URL 인코딩, 기타 인코딩 등으로 탐지 룰을 우회하고, 악성 페이로드가 정상 문자열처럼 보이도록 은닉한다.                                                                                                              |
| Validation               | curl 등으로 페이로드를 실행해 HTTP 응답 코드·본문·에러 메시지를 비교·분석하여 SQL Smuggling 성공 여부를 확인한다.                                                                                                                       |
| Injection & Exploitation | 검증된 페이로드를 실제 HTTP 요청, 파라미터에 넣어서 DB 서버에서 동적 SQL로 실행시켜 권한 상승, 테이블 조작 등의 공격을 수행한다.                                                                                                                   |
| Exfiltration             | Error-based, Union-based, Time-based 기법 등으로 공격한 뒤 응답 값을 통해 데이터를 획득한다.                                                                                                                             |

<div style="text-align: left;">
  <img
    src="/sql_smuggling.png"
    alt="SQL Smuggling 다이어그램"
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
      <td>Enumerate Platform Syntax</td>
      <td><code>\' → 필터 결과 \'' 확인</code></td>
      <td>DB별 이스케이프 규칙을 식별해 필터가 놓치는 패턴을 찾는다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Map Signature Blacklists</td>
      <td><code>“UNION SELECT” 등 금지어 탐지 여부 테스트</code></td>
      <td>시그니처 기반 차단 목록을 파악해 이후 우회 전략의 기준을 마련한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Desync

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
      <td>Trigger Best-fit Translation</td>
      <td><code>name=%CA%BCadmin%CA%BC</code></td>
      <td>애플리케이션 파서는 인식하지 못하는 U+02BC가 DB 파서에서 '로 변환되어 문자열 경계가 무너진다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Abuse Backslash Escaping</td>
      <td><code>...\'); (MySQL \ 이스케이프)</code></td>
      <td>애플리케이션이 \' 이스케이프 시퀀스를 제거하지 않는 점을 악용해, 백엔드 DB 파서가 이스케이프를 적용해 원래의 따옴표로 인식하도록 만든다.</td>
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
      <td>Collide Double-Escape Sequences</td>
      <td><code>입력 \' → 필터 후 \'</code></td>
      <td>필터가 추가한 따옴표가 DB에서 실제 따옴표로 남아 쿼리 구조를 변경한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Build SQL via CHAR()/EXEC()</td>
      <td><code>CHAR(0x55,0x4E,0x49...), EXEC('INS'+'ERT ...')</code></td>
      <td>문자열 분해·결합으로 검증을 피하고 DB 단계에서만 완전한 명령으로 재조립된다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Layer Multi-Encodings</td>
      <td><code>%255c%2527 → 디코딩 후 \'</code></td>
      <td>다단 인코딩을 겹쳐 검증/DB 디코딩 차이를 이용해 원문 토큰을 복원시킨다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Obfuscation

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
      <td>Inject Inline Comments</td>
      <td><code>UN/**/ION SE/**/LECT</code></td>
      <td>키워드를 주석으로 분절해 시그니처 매칭을 무력화한다. DB는 주석을 제거하고 실행한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Expand Whitespace Tokens</td>
      <td><code>SEL ECT, \nUNION\nSELECT</code></td>
      <td>과도한 공백·개행으로 토큰화를 흐려 탐지를 우회한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Switch Encoding Forms</td>
      <td><code>%27, 0x27 등 혼합 사용</code></td>
      <td>서로 다른 인코딩 표현으로 문자열 일치 검사를 회피하고 DB에서만 복원되게 한다.</td>
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
      <td>Execute SP and Observe Breakout</td>
      <td><code>exec(@s) 포함 SP에 %CA%BC 전달</code></td>
      <td>결과/에러로 변환된 따옴표가 쿼리를 깨뜨렸는지 확인한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Compare Encoded vs Plain Inputs</td>
      <td><code>URL/Hex/Unicode 변형 vs 원본 응답 비교</code></td>
      <td>다른 표현이 동일 SQL로 실행되면 우회 성공으로 판단한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

### Injection & Exploitation

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
      <td>Run Arbitrary Dynamic SQL</td>
      <td><code>'; DROP TABLE dataTable--</code></td>
      <td>동적 SQL 영역을 장악해 임의 명령을 실행한다.</td>
      <td>[1]</td>
    </tr>
    <tr>
      <td>Break Out of Quoted Params</td>
      <td><code>U+02BC...U+02BCU+02BC OR 1=1--</code></td>
      <td>변환된 따옴표로 문자열을 탈출해 WHERE 절을 조작한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>



## 분류에 해당하는 문서

\[1] [https://dl.packetstormsecurity.net/papers/database/SQL\_Smuggling.pdf](https://dl.packetstormsecurity.net/papers/database/SQL_Smuggling.pdf)

## 기타 참고문헌

---
title: Double Dash SQL Injection
description: 
published: true
date: 2025-07-31T21:25:33.740Z
tags: target environment analysis, line comment creation, multi‑line string injection, prepared statement bypass, syntax error confirmation, multiline comment injection, stacked‑query exfiltration
editor: markdown
dateCreated: 2025-07-25T10:04:49.612Z
---

# Double Dash SQL Injection

## 개요

“Double Dash SQL Injection”은 SQL 주석 기호 `--` 뒤에 공백이 없는 경우, 데이터베이스 파서에 따라 이를 주석으로 처리할지 여부가 달라지는 구조적 해석 차이를 악용하는 기법이다. 일부 파서는 `--payload` 형식을 전체 주석으로 간주하고 이후 구문을 무시하지만, 다른 파서는 공백이 없으면 주석으로 인식하지 않고 해당 문자열을 코드로 처리한다. 이로 인해 동일한 입력이 구문 트리 수준에서 상이하게 해석되며, WAF나 프록시 등 보안 필터는 이를 무해한 주석으로 판단한 반면, 백엔드는 유효한 쿼리로 실행하는 흐름이 형성된다. 결과적으로 구문 해석 불일치로 인해 SQL 인젝션 탐지 우회 및 공격 흐름 삽입이 가능해진다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)
> → 구조 해석 불일치 (Structural Interpretation Mismatch)
> → Double Dash SQL Injection

## 절차

| Phase          | Description                                                                                             |
| -------------- | ------------------------------------------------------------------------------------------------------- |
| Reconnaissance | 대상이 PostgreSQL, SQLite 등 – 뒤에 공백이 필요없는 DB를 사용하는지 파악하고, Simple Query Mode로 동작하는 클라이언트 라이브러리를 사용하는지 파악한다. |
| Injection      | 클라이언트가 음수의 숫자 값을 사용하였을 때 (- 를 사용할 때) SQL을 해석하는 과정에서 `--`를 라인 주석으로 잘못 인식하는 점을 이용하여 SQL 구문을 삽입한다.         |
| Obfuscation    | 변형한 문자열 앞뒤로 공백, TAB, 인코딩 등을 섞어 필터를 우회하거나, 계속해서 멀티라인 문자열을 삽입하여 다음 구문까지 이어 붙인다.                           |
| Bypass         | WAF 탐지를 피하기 위하여 URL‑encoding, Chunked‑bodyd 등을 사용한다.                                                    |
| Validation     | 오류, 반환값, 서버 로그로 주석 생성 및 구문이 변경되었는지 확인한다.                                                                |
| Exploitation   | 권한 상승, 데이터 변조, 추가 쿼리 실행 등 원하는 악성 SQL을 수행한다.                                                             |
| Exfiltration   | `SELECT`문 등으로 민감 데이터를 유출한다.                                                                             |

<div style="text-align: left;">
  <img
    src="/double_dash_sql_injection.png"
    alt="Double Dash SQL Injection 다이어그램"
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
      <td>Target Environment Analysis</td>
      <td><code> PgJDBC &lt; 42.7.2</code></td>
      <td>대상 시스템이 사용하는 데이터베이스와 클라이언트 라이브러리의 종류 및 버전을 식별한다. Extended Query Mode를 사용하지 않고 클라이언트 측에서 파라미터를 직접 쿼리문에 삽입하는 ‘단순 쿼리 프로토콜’을 사용하는지 파악하여 공격 가능성을 정찰한다.</td>
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
      <td>Line Comment Creation</td>
      <td> <pre> <code> UPDATE accounts SET balance = balance - ?</code> </pre> with <pre><code> $1 = -42 </code></pre> </td>
      <td>숫자형 파라미터에 음수 값을 주입하여, 클라이언트 라이브러리가 쿼리를 생성하는 과정에서 <code>--</code> 시퀀스가 만들어지도록 유도한다. PostgreSQL과 같은 데이터베이스는 이를 라인 주석의 시작으로 해석하여, 이후의 쿼리 구문을 무력화시킨다.</td>
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
      <td>Multi-line String Injection</td>
      <td>Account ID parameter = <code> 'foo\nbar'</code> </td>
      <td>PostgreSQL과 같이 여러 줄의 문자열 리터럴을 지원하는 데이터베이스의 특성을 이용한다. 주석 생성 후 이어지는 문자열 파라미터에 개행 문자(\n)를 포함시켜, 다음 줄에 새로운 SQL 구문을 삽입할 수 있도록 페이로드를 구성한다.</td>
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
      <td>Prepared Statement Bypass</td>
      <td><code> SELECT 1-? </code>  with <code> $1 = -1 </code> </td>
      <td>Prepared Statement(매개변수화된 쿼리)를 우회한다. 취약한 클라이언트 라이브러리가 서버에 쿼리를 보내기 전, 안전하지 않은 방식으로 파라미터를 삽입하는 과정의 허점을 이용하므로, 일반적인 SQL 인젝션 방어 기법이 무력화된다.</td>
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
      <td>Syntax Error Confirmation</td>
      <td><code> UPDATE ... </code> query results in a syntax error.</td>
      <td>주입된 페이로드로 인해 완성된 쿼리가 의도적으로 유효하지 않은 구문이 되도록 만든다. 서버로부터 구문 오류(Syntax Error) 응답을 수신함으로써, 쿼리의 구조가 성공적으로 조작되었음을 검증한다.</td>
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
      <td>Multiline Comment Injection</td>
      <td>
        <pre><code># Parameterized query:
SELECT * FROM example WHERE result=-$1 OR name=$2;<br>
# Parameter values:<br>
<br>
# $1: -42<br>
# $2: "foo\n 1 AND 1=0 UNION SELECT * FROM secrets; --"<br>
<br>
# Resulting query after preparation:
SELECT * FROM example WHERE result=--42 OR name= 'foo<br> 
1 AND 1=0 UNION SELECT * FROM secrets; --';<br>
# Resulting query with comments stripped:SELECT * FROM example WHERE result= 
1 AND 1=0 UNION SELECT * FROM secrets;</code></pre>
      </td>
      <td>-$1 자리에 -42를 넣으면 --42가 되어 SQL 주석 시작으로 인식되고 이후 문자열 파라미터에 개행 문자(\n)를 포함 시켜 주석을 끝내게 한 뒤 UNION 구문을 삽입하여 민감 데이터를 획득한다.</td>
      <td>[A]</td>
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
      <td>Stacked‑Query Exfiltration</td>
      <td><pre> <code> UPDATE account SET balance=balance--1 WHERE account_id='foo'; INSERT INTO users (name, role, password) VALUES ($$attacker$$, $$admin$$, $$…$$);-- ';</code> </pre> </td>
      <td>멀티라인 주입으로 -- 주석처리 한 뒤 세미콜론(;)으로 첫 쿼리를 종료하고 이어서 <code> INSERT INTO users </code>  구문을 실행해 공격 계정을 추가하거나 권한을 탈취한다.</td>
      <td>[1]</td>
    </tr>
  </tbody>
</table>

## 분류에 해당하는 문서

\[1], https://www.sonarsource.com/blog/double-dash-double-trouble-a-subtle-sql-injection-flaw/ 
\[A] https://github.com/vitaly-t/pg-promise/discussions/911

## 기타 참고문헌

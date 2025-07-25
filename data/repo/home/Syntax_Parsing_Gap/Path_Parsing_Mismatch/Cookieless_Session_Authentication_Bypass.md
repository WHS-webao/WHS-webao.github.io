---
title: Cookieless Session Authentication Bypass
description: 
published: true
date: 2025-07-25T17:02:13.596Z
tags: 
editor: markdown
dateCreated: 2025-07-23T10:44:10.847Z
---

# Cookieless Session Authentication Bypass

## 개요
“Cookieless Session Authentication Bypass”는 ASP.NET 프레임워크에서 경로 기반 세션 식별자인 `/(S(sessionid))/` 형식을 사용하는 환경에서, URL 경로 파싱 방식에 따라 인증 우회 또는 소스 코드 노출이 발생할 수 있는 취약점이다. 공격자는 세션 식별자를 포함한 경로를 조작하거나 중복된 경로 요소를 삽입하여, ASP.NET 내부의 요청 매핑 로직과 인증 검증 로직 간의 파싱 불일치를 유발한다. 그 결과, 인증이 요구되지 않는 위치로 redirection되거나, 정적 파일 경로로 잘못 해석되어 `.aspx` 등의 서버 파일이 소스 코드로 노출될 수 있다. 이로 인해 인증 우회, 권한 상승, 애플리케이션 내부 구조 유출 등 다양한 보안 위협이 발생한다.

## 분류

> 구문 분석 불일치 (Syntax Parsing Gap)  
> → 경로 파싱 불일치 (Path Parsing Mismatch)  
> → Cookieless Session Authentication Bypass

## 절차

| Phase            | Description                                                                                                                                                                                        |
|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Reconnaissance   | 대상 웹 애플리케이션이 cookieless 세션을 지원하는지 확인하고 `/(S(...))/` 패턴이 사용되는지 탐지하고 인증, 필터링이 적용된 경로를 수집한다.                                                           |
| Divergence       | 중첩된 세션 세그먼트(`/(S(X))/`) 삽입으로 IIS 파서와 ASP.NET 런타임 간 세션 경계 파싱 차이를 유도한다.                                                                                                |
| Obfuscation      | URL 인코딩, 8.3 짧은 이름, `::$INDEX_ALLOCATION` 트릭 등을 활용해 WAF 및 경로 검사 룰을 우회한다.                                                                                                    |
| Injection & Bypass | `/(S(X))/` 토큰을 정상 경로 사이에 주입해서 인증, 경로 필터를 우회한다. `Bin` 폴더 요청이 ASP.NET 모듈로 라우팅 되도록 유도한다.                                                                     |
| Validation       | Protected 페이지와 DLL 경로에 대해 `curl -I`/`curl --range` 요청을 보내 200 OK 및 DLL 바이너리 응답 여부를 확인한다.                                                                               |
| Exploitation     | 인증 없이 보호 리소스에 접근하여 anonymous 권한으로 페이지를 획득하고, 보호 리소스(`.aspx`,DLL)을 직접 실행하도록 시도하고, `.ashx`/`.asmx` 핸들러에 DLL 로드를 유도한다.                              |
| Exfiltration     | DLL을 다운로드 후 디컴파일·역공학을 통해 소스 코드를 획득한다.                                                                                                                                      |

<div style="text-align: left;">
  <img
    src="/cookieless_session_authentication_bypass.png"
    alt="Cookieless Session Authentication Bypass 다이어그램"
    width="1000"
    style="display: block;
           margin: 0 0 1em 0;
           height: auto;"
  />
</div>

## 공격기법

### Reconnaissance

| 공격 이름                     | 예시 페이로드                                      | 설명                                                                                                                         | 출처 |
|-------------------------------|----------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|------|
| Cookieless Session Discovery  | `/default.aspx/(S(dummy))/profile.aspx`            | URL 경로에 `/(S(...))/` 세션 토큰 패턴을 삽입하여 cookieless session 지원 여부를 탐지한다. 이를 통해 공격 가능성을 사전 확인한다. | [1]  |

### Divergence

| 공격 이름                          | 예시 페이로드                                                      | 설명                                                                                                                                  | 출처 |
|------------------------------------|---------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|------|
| Double Session Token Injection     | `/public/(S(abc123))/page/(S(xyz456))/dashboard.aspx`               | IIS 인증 모듈과 ASP.NET 라우팅 모듈 간 파싱 경계 차이를 유발하기 위해 경로 중간에 이중 `/(S(...))/` 세그먼트를 삽입하여 인증 필터를 우회한다. | [1]  |
| Application Pool Confusion Injection | `/(S(X))/(S(X))/classic/AppPoolPrint.aspx`                          | 경로 시작 부분에 이중 `/(S(X))/` 세그먼트를 주입하여, IIS가 해당 요청을 자식 애플리케이션이 아닌 부모의 Application Pool 컨텍스트에서 처리하도록 유도한다. | [1]  |

### Obfuscation

| 공격 이름          | 예시 페이로드                                          | 설명                                                                                     | 출처 |
|--------------------|-------------------------------------------------------|------------------------------------------------------------------------------------------|------|
| NTFS ADS Evasion   | `/bin/SharpApp.dll::$INDEX_ALLOCATION`                | NTFS Alternate Data Stream(ADS) 기능을 활용해 URL 필터링 및 WAF 룰을 우회하여 DLL 경로 노출을 유도한다. | [2]  |

### Injection & Bypass

| 공격 이름                       | 예시 페이로드                | 설명                                                                                                                                     | 출처 |
|---------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------|------|
| Path Injection Auth Bypass      | `/protected/(S(evil))/page.aspx` | `(S(...))` 토큰을 경로에 삽입하여 IIS가 토큰을 무시하도록 유도하고, ASP.NET 모듈이 이를 유효 세션으로 처리하게 만들어 인증 우회에 성공한다. | [1]  |
| ASP.NET Module Routing Exploit  | `/bin/SharpApp.dll`          | cookieless 세션 토큰 없이도 `/bin/*.dll` 요청이 ASP.NET 파이프라인으로 라우팅되도록 유도하여 DLL 파일을 노출한다.                          | [2]  |

### Validation

| 공격 이름                      | 예시 페이로드                                                                    | 설명                                                                                                 | 출처 |
|--------------------------------|---------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|------|
| Anonymous Access Verification  | `curl -I https://victim.com/(S(session))/protected/resource.aspx`                | 인증 없이 보호된 리소스에 요청을 보내고, HTTP 200 OK 수신 여부를 통해 인증 우회 성공 여부를 검증한다. | [1]  |
| DLL Exposure Validation        | `curl --range 0-1023 https://victim.com/bin/SharpApp.dll`                       | Range 헤더로 DLL 파일 일부를 요청해 응답에 MZ 바이너리 헤더가 포함되었는지 확인함으로써 DLL 노출 여부를 확인한다. | [2]  |

### Exploitation

| 공격 이름                          | 예시 페이로드                                                      | 설명                                                                                                                       | 출처 |
|------------------------------------|-------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|------|
| Anonymous Resource Execution       | `/webform/(S(exec))/handler.ashx`                                | 익명 권한으로 `.ashx` 또는 `.asmx` 핸들러에 접근하여 서버 측 로직을 실행하고, 동적 로드된 DLL을 통해 추가 권한 상승을 시도한다. | [1]  |
| PathInfo-based App Pool Confusion  | `/cla/(S(X))ssic/AppPoolPrint.aspx/(S(X))/`                      | ASP.NET의 PathInfo 기능 중 `.aspx` 확장자 뒤에 `/(S(X))/` 패턴을 추가하여 부모 애플리케이션 풀 컨텍스트에서 페이지가 실행되도록 유도한다. | [1]  |

### Exfiltration

| 공격 이름                              | 예시 페이로드            | 설명                                                                                                            | 출처 |
|----------------------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------|------|
| Source Code Extraction via DLL Decompilation | `/bin/SharpApp.dll`     | 다운로드한 DLL 파일을 디컴파일 도구(dnSpy 등)로 분석하여 내부 비즈니스 로직 및 소스 코드를 추출한다.             | [2]  |

---

## 분류에 해당하는 문서
- [1] [Cookieless DuoDrop: IIS Auth Bypass & App Pool Privesc in ASP.NET Framework (CVE-2023-36899 & CVE-2023-36560)](https://soroush.me/blog/2023/08/cookieless-duodrop-iis-auth-bypass-app-pool-privesc-in-asp-net-framework-cve-2023-36899/)
- [2] [Source Code Disclosure in ASP.NET apps](https://swarm.ptsecurity.com/source-code-disclosure-in-asp-net-apps/)
  

## 기타 참고 문헌
- [a] https://learn.microsoft.com/en-us/previous-versions/dotnet/articles/aa479314(v=msdn.10)  
- [b] https://blog.isec.pl/all-is-xss-that-comes-to-the-net/  
- [c] https://learn.microsoft.com/en-us/archive/msdn-magazine/2009/march/security-briefs-protect-your-site-with-url-rewriting  
- [d] https://www.sans.org/blog/session-attacks-and-asp-net-part-2/  

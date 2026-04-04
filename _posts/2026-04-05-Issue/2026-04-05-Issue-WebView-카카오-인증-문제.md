---
title: 카카오 인증 후 복귀 시 Page Load Failure 문제 해결
author: Jooye0n
date: 2024-03-13 18:07:00 +09:00
categories: [Issue]
tags: [webview, kakao, polling, javascript, hybrid-app]
toc: true
toc_sticky: true
toc_label: 목차
math: true
mermaid: true
---

## 1. 문제 상황

-   스타기업뱅킹 앱 (Hybrid App, WebView) 내에서 전자서명 진행
-   카카오 인증을 위해 외부 앱 (카카오톡) 호출
-   인증 완료 후 앱으로 복귀하는 구조

### 발생 문제

1.  Android에서 카카오 인증 후 복귀 시 일부 사용자에게서 Page Load Failure 발생
2.  인증수단 변경 시 (카카오 → KB 인증서)
    -   기존 카카오 폴링이 계속 돌고
    -   KB 인증 폴링도 동시에 수행됨

### 실제 환경

-   WebView 기반 하이브리드 앱
-   JS에서 location.href로 카카오톡 scheme 호출
-   setTimeout 기반 비동기 호출 사용
-   인증 상태 조회는 polling 방식 (setTimeout 재귀 호출)

------------------------------------------------------------------------

## 2. 원인 분석

### WAS:

-   별도 문제 없음 (정상 응답)
-   API 호출 중복 발생은 있었으나 서버 로직 문제는 아님

### DB:

-   중복 polling으로 인해 동일 거래 조회 API 다건 호출 발생
-   부하 증가 가능성 존재

### 코드:

#### 1) WebView + 카카오 인증 문제

``` javascript
setTimeout(() => {
  location.href = schemeUrl;
}, 100);
```

👉 문제 - Android WebView에서는 user gesture 없이 외부 앱 호출 시 제한
발생 - setTimeout 내부 호출 → 비동기 실행 → 사용자 액션으로 인식되지
않음 - 결과적으로 WebView lifecycle 깨짐 또는 페이지 초기화

------------------------------------------------------------------------

#### 2) 폴링 중복 문제

``` javascript
function poll() {
  setTimeout(() => {
    checkStatus();
    poll();
  }, 500);
}
```

👉 문제 - 카카오 인증 polling 시작 - 중간에 인증수단 변경 - 기존 polling
종료 없이 새로운 polling 시작

결과: - 카카오 + KB polling 동시에 실행 - 상태 꼬임 및 불필요 API 호출
발생

------------------------------------------------------------------------

## 3. 해결 방법

### 1) WebView 외부 앱 호출 방식 수정

#### 기존

``` javascript
setTimeout(() => {
  location.href = schemeUrl;
}, 100);
```

#### 변경

``` javascript
location.href = schemeUrl;
```

👉 반드시 사용자 클릭 이벤트 내부에서 즉시 실행

------------------------------------------------------------------------

### 2) polling 제어 로직 추가

#### 상태 관리 변수 도입

``` javascript
let currentAuthType = null;
let pollingActive = false;
```

#### polling 시작 시

``` javascript
function startPolling(type) {
  currentAuthType = type;
  pollingActive = true;
  poll(type);
}
```

#### polling 실행

``` javascript
function poll(type) {
  if (!pollingActive || currentAuthType !== type) return;

  setTimeout(() => {
    checkStatus(type);
    poll(type);
  }, 500);
}
```

#### 인증수단 변경 시

``` javascript
function changeAuthType(newType) {
  pollingActive = false;
  currentAuthType = newType;
}
```

👉 핵심 - 단일 polling만 유지 - 인증수단 변경 시 기존 polling 즉시 중단

------------------------------------------------------------------------

## 4. 결과

-   Android WebView에서 카카오 인증 후 페이지 깨짐 현상 해결
-   외부 앱 호출 안정성 확보
-   polling 중복 제거
-   API 호출 횟수 감소
-   인증 상태 꼬임 문제 해결

------------------------------------------------------------------------

## 5. 배운 점

-   WebView는 일반 브라우저와 다르게 user gesture 정책 영향을 강하게
    받는다
-   setTimeout / Promise 내부에서 외부 앱 호출은 위험하다
-   polling 구조는 반드시 중단 조건과 상태 관리가 필요하다
-   인증/결제 같은 흐름에서는 단일 상태 관리 (state control)가 핵심이다

------------------------------------------------------------------------

## 6. 추가 정리

-   Android WebView는 Chrome 정책을 따르며 외부 앱 호출 시 제약 존재
-   Hybrid App에서는 JS 로직보다 앱 lifecycle 고려가 더 중요
-   polling 대신 가능하면 event 기반 구조가 더 안정적

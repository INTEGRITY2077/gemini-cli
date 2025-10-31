---
title: Chat Interface Overview
slug: chat-interface-overview
---

> **문서 링크**: https://geminicli.com/docs/chat-interface-overview

# Gemini CLI 대화 인터페이스 개요

## 전체 구조

- **엔트리포인트**: `packages/cli/src/gemini.tsx`의 `startInteractiveUI`가 Ink
  렌더러를 초기화하고 `SettingsContext`, `KeypressProvider`,
  `SessionStatsProvider`, `VimModeProvider` 등을 중첩하여 `AppContainer`를
  렌더링합니다. 또한 버전 확인, 업데이트 알림, 종료 처리 등 부트스트랩 작업을
  수행합니다.
- **전역 상태 허브**: `packages/cli/src/ui/AppContainer.tsx`는 히스토리, 테마,
  인증, 터미널 크기, 메시지 큐 등 주요 상태를 결합하고 `useGeminiStream`을
  호출하여 스트리밍 상태와 사고(thought) 로그를 수집한 뒤 `UIStateContext`와
  `UIActionsContext`로 하위 UI에 제공합니다.

## 스트리밍 및 메시지 처리

- **스트림 오케스트레이션**: `packages/cli/src/ui/hooks/useGeminiStream.ts`는
  슬래시/＠ 명령, 셸 명령, 일반 쿼리를 분기하고 Gemini API 호출, 툴 실행
  스케줄링, 취소 및 에러 핸들링을 담당합니다. 스트림 이벤트(콘텐츠, 인용, 종료
  등)를 받아 히스토리를 갱신하며 긴 응답은 정적 영역으로 밀어 렌더링 부하를
  줄입니다.
- **메시지 제출 흐름**: `submitQuery` 로직이 동시 전송을 차단하고 프롬프트 ID를
  생성해 로깅 및 루프 감지에 활용합니다. 처리 결과에 따라 히스토리 반영, 재시도,
  사용자 취소 등의 후처리를 수행합니다.

## UI 레이아웃

- **최상위 레이아웃**: `packages/cli/src/ui/App.tsx`는 종료 화면 여부와
  스크린리더 옵션에 따라 기본 레이아웃과 접근성 모드를 전환하고, 스트리밍 상태를
  `StreamingContext`로 배포합니다.
- **기본 레이아웃 구성**: `packages/cli/src/ui/layouts/DefaultAppLayout.tsx`는
  메인 대화 영역(`MainContent`), 알림/다이얼로그/컴포저 섹션, 종료 경고 등을
  배치하며 터미널 깜빡임 완화를 위한 훅을 적용합니다.
- **히스토리 렌더링**: `packages/cli/src/ui/components/MainContent.tsx`는 확정된
  히스토리를 `<Static>`으로 고정 렌더링하고, 스트리밍 중인 메시지는 별도의
  오버플로우 영역에서 관리합니다.

## 사용자 입력 경험

- **컴포저 조립**: `packages/cli/src/ui/components/Composer.tsx`는 로딩
  인디케이터, 컨텍스트 요약, 자동 승인/셸 모드 배지, 큐 메시지, 디버그 패널,
  입력 프롬프트 등을 통합해 하단 입력 영역을 구성합니다.
- **입력 프롬프트**: `packages/cli/src/ui/components/InputPrompt.tsx`는 텍스트
  버퍼, 히스토리 역검색, 자동완성, Vim 모드, 붙여넣기 보호 등을 제공하며 제출 시
  상위 `onSubmit`으로 메시지를 전달합니다. 스트리밍 중에는 커맨드 큐잉을
  조절하여 UI 일관성을 유지합니다.

## 난독화된 번들 및 원본 소스

- **번들 위치**: `bundle/gemini.js`는 배포용으로 번들링된 파일로, 변수명이
  압축되어 읽기 어렵습니다.
- **가독성 높은 원본**: 위에서 언급한 TypeScript 파일들은 모두
  `packages/cli/src` 아래에 위치하며 모듈 경계와 import 구조가 유지되어 있어
  대화 인터페이스의 작동 원리를 분석하기 쉽습니다.

## 참고 경로 요약

- 대화형 CLI 루트: `packages/cli`
- UI 소스 트리: `packages/cli/src/ui`
- 스트리밍 훅: `packages/cli/src/ui/hooks/useGeminiStream.ts`
- 입력 관련 컴포넌트: `packages/cli/src/ui/components`
- 레이아웃 관련 파일: `packages/cli/src/ui/layouts`

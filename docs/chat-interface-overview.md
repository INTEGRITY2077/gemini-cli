---
title: Chat Interface Overview
slug: chat-interface-overview
---

> **문서 링크**: https://geminicli.com/docs/chat-interface-overview

# Gemini CLI 대화 인터페이스 개요

본 문서는 CLI에서 사용자가 Gemini와 대화할 때 어떤 컴포넌트가 어느 위치에 있고,
어떻게 서로 연결되는지를 **코드 예시와 함께** 설명합니다. 아래 각 절의 경로를
따라가면 동일한 로직을 직접 확인하고 복사해갈 수 있습니다.

## 1. 전체 진입 흐름

`packages/cli/src/gemini.tsx`의 `startInteractiveUI`가 Ink 렌더러를 띄우면서
모든 전역 컨텍스트를 한 번에 준비합니다. `SettingsContext`, `KeypressProvider`,
`SessionStatsProvider`, `VimModeProvider`를 중첩하여 `AppContainer`가 필요로
하는 상태를 주입하고, 렌더 전에 버전 조회·윈도우 타이틀 설정·업데이트 검사 같은
부트스트랩 작업을 마칩니다.

```tsx title="packages/cli/src/gemini.tsx (발췌)"
const AppWrapper = () => {
  const kittyProtocolStatus = useKittyKeyboardProtocol();
  return (
    <SettingsContext.Provider value={settings}>
      <KeypressProvider
        kittyProtocolEnabled={kittyProtocolStatus.enabled}
        config={config}
        debugKeystrokeLogging={settings.merged.general?.debugKeystrokeLogging}
      >
        <SessionStatsProvider>
          <VimModeProvider settings={settings}>
            <AppContainer
              config={config}
              settings={settings}
              startupWarnings={startupWarnings}
              version={version}
              initializationResult={initializationResult}
            />
          </VimModeProvider>
        </SessionStatsProvider>
      </KeypressProvider>
    </SettingsContext.Provider>
  );
};
```

## 2. AppContainer: 상태 및 컨텍스트 결합

`packages/cli/src/ui/AppContainer.tsx`는 CLI UI의 허브입니다. 설정, 인증 상태,
히스토리, 테마, 확장 업데이트, 터미널 치수, 메시지 큐 등을 모아서 하나의
`uiState` 객체로 묶고 `UIStateContext`로 배포합니다. 동시에 메시지 제출·화면
초기화·탈출 프롬프트 같은 액션은 `UIActionsContext`로 분리해 하위 컴포넌트가
필요한 콜백만 가져가도록 합니다.

```tsx title="packages/cli/src/ui/AppContainer.tsx (발췌)"
const uiState: UIState = useMemo(
  () => ({
    history: historyManager.items,
    pendingHistoryItem,
    pendingToolCallGroupDisplay,
    streamingState,
    thought,
    isProcessing,
    shellModeActive,
    embeddedShellFocused,
    // ...생략: 테마, 모델, 프라이버시 알림 등 기타 상태
  }),
  [
    historyManager.items,
    pendingHistoryItem,
    pendingToolCallGroupDisplay,
    streamingState,
    thought,
    isProcessing,
    shellModeActive,
    embeddedShellFocused,
    // ...
  ],
);

const uiActions: UIActions = useMemo(
  () => ({
    submitUserMessage: submitMessage,
    clearScreen,
    toggleShellMode,
    openPermissionsDialog,
    closePermissionsDialog,
    // ...생략: Vim 토글, 디버그 패널 제어 등
  }),
  [
    submitMessage,
    clearScreen,
    toggleShellMode,
    openPermissionsDialog,
    closePermissionsDialog,
    // ...
  ],
);

return (
  <UIStateContext.Provider value={uiState}>
    <UIActionsContext.Provider value={uiActions}>
      <ConfigContext.Provider value={config}>
        <AppContext.Provider value={appContextValue}>
          <ShellFocusContext.Provider value={shellFocusValue}>
            <App {...props} />
          </ShellFocusContext.Provider>
        </AppContext.Provider>
      </ConfigContext.Provider>
    </UIActionsContext.Provider>
  </UIStateContext.Provider>
);
```

이 과정에서 `useMessageQueue`는 전송 중 추가 입력을 큐에 보관했다가 빈틈이
생기면 버퍼로 되돌리고, `useTextBuffer`는 재시도 시 같은 내용을 복원합니다.
따라서 긴 대화에서도 입력 일관성이 유지됩니다.

## 3. Gemini 스트림 파이프라인

실제 채팅 엔진은 `packages/cli/src/ui/hooks/useGeminiStream.ts`가 담당합니다.
슬래시 명령과 `@` 명령, 셸 명령을 먼저 판별해 필요한 경우 툴 호출만 예약하고,
일반 메시지는 히스토리에 즉시 추가한 뒤 Gemini API로 전송할 페이로드를 만듭니다.

```tsx title="packages/cli/src/ui/hooks/useGeminiStream.ts (발췌)"
export const useGeminiStream = (
  geminiClient,
  history,
  addItem,
  config,
  settings,
  onDebugMessage,
  handleSlashCommand,
  shellModeActive,
  getPreferredEditor,
  onAuthError,
  performMemoryRefresh,
  modelSwitchedFromQuotaError,
  setModelSwitchedFromQuotaError,
  onEditorClose,
  onCancelSubmit,
  setShellInputFocused,
  terminalWidth,
  terminalHeight,
  isShellFocused?,
) => {
  const [isResponding, setIsResponding] = useState(false);
  const [pendingHistoryItem, , setPendingHistoryItem] =
    useStateAndRef<HistoryItemWithoutId | null>(null);
  const [toolCalls, scheduleToolCalls] = useReactToolScheduler(...);
  const pendingToolCallGroupDisplay = useMemo(
    () => (toolCalls.length ? mapTrackedToolCallsToDisplay(toolCalls) : undefined),
    [toolCalls],
  );

  const submitQuery = useCallback(async (parts: PartListUnion) => {
    if (isResponding) {
      onCancelSubmit();
      return;
    }
    // 1) 히스토리 업데이트
    // 2) Gemini API 스트림 생성
    // 3) 스트림 이벤트(Content/Quote/Thought 등) 처리
    // 4) 오류 및 취소 시 정리
  }, [isResponding, onCancelSubmit]);

  return {
    submitQuery,
    pendingHistoryItem,
    pendingToolCallGroupDisplay,
    // ...생략: streamingState, thought, loop detection 등
  };
};
```

스트림 이벤트는 콘텐츠 조각, 사용자 취소, 에러, 인용, 종료 등으로 세분화되어
히스토리에 반영됩니다. 긴 응답은 `findLastSafeSplitPoint`로 잘라 `Static` 영역에
배치해 잦은 리렌더링을 줄입니다. 또한 툴 PTY 식별자 추적을 통해 셸이나 도구가
실행 중일 때 `StreamingState`를 갱신하고, UI는 이를 바탕으로 로딩 인디케이터와
입력 비활성화를 맞춥니다.

## 4. UI 레이아웃 및 렌더링

- **`packages/cli/src/ui/App.tsx`**: 종료 메시지 여부와 스크린리더 모드를 확인해
  기본 레이아웃 또는 접근성 레이아웃을 선택하고, 현재 스트리밍 상태를
  `StreamingContext`로 내려 하위 컴포넌트가 인디케이터를 조정하게 합니다.
- **`packages/cli/src/ui/layouts/DefaultAppLayout.tsx`**: 상단에는 히스토리/정적
  영역(`MainContent`)을, 하단에는 알림·다이얼로그·컴포저를 배치하고 Flicker
  detector 훅으로 터미널 깜빡임을 줄입니다.
- **`packages/cli/src/ui/components/MainContent.tsx`**: 과거 히스토리는
  `<Static>`으로 고정 렌더링하며 아직 확정되지 않은 메시지는 별도의 Overflow
  영역에서 관리해 스트리밍 중인 항목만 갱신합니다.

## 5. 입력 프롬프트와 사용자 상호작용

하단 입력 영역은 `Composer`와 `InputPrompt` 조합으로 완성됩니다.

- **`Composer` (`packages/cli/src/ui/components/Composer.tsx`)**: 로딩
  인디케이터, 컨텍스트 요약, 자동 승인/셸 모드 배지, 큐 메시지, 디버그 패널,
  입력 프롬프트를 모아 UIActions에서 받은 콜백과 상태를 하위 컴포넌트에
  전달합니다.
- **`InputPrompt` (`packages/cli/src/ui/components/InputPrompt.tsx`)**: 텍스트
  버퍼, 히스토리 역검색, 슬래시/＠ 자동완성, Vim 모드, 붙여넣기 보호, 승인 모드
  등을 지원합니다. 메시지 제출이 진행 중이면 입력을 큐에 쌓았다가 비어 있는
  타이밍에 다시 버퍼로 복원해 사용자가 입력을 잃지 않도록 합니다.

이 모든 입력 훅은 `AppContainer`가 계산한 입력 폭/제안 폭을 기준으로 동작해
터미널 리사이즈 후에도 일관된 레이아웃을 유지합니다.

## 6. 난독화된 번들과 원본 소스의 관계

- **번들 위치**: 배포 파일 `bundle/gemini.js`는 이름이 축약되어 있어 직접 수정이
  어렵습니다.
- **원본 위치**: 위에서 언급한 TypeScript 소스(예: `AppContainer.tsx`,
  `useGeminiStream.ts`)는 모두 `packages/cli/src/**` 아래에 있으며, 모듈 경계와
  주석이 유지되어 있어 동작 원리를 추적하기 쉽습니다.

## 7. 빠른 경로 요약

- 대화형 CLI 루트: `packages/cli`
- UI 소스 트리: `packages/cli/src/ui`
- 스트리밍 훅: `packages/cli/src/ui/hooks/useGeminiStream.ts`
- 입력 관련 컴포넌트: `packages/cli/src/ui/components`
- 레이아웃 관련 파일: `packages/cli/src/ui/layouts`
- 번들 산출물: `bundle/gemini.js`

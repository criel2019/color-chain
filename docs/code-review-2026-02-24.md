# ColorChain Client Code Review
Last Updated: 2026-02-24

---

## Executive Summary

`index.html` 전체(약 1,802줄)를 검토했습니다. 코드는 전반적으로 잘 구조화되어 있으며, 이번 세션에서 추가된 자동재연결 로직과 REMATCH UI는 의도한 동작을 달성하고 있습니다. 다만 일부 보안 취약점, 런타임 안정성 문제, 설계상 주의가 필요한 지점이 발견되었습니다.

---

## [CRITICAL] 치명적 문제

### C-1. WebSocket 메시지 역직렬화 시 서버 데이터 무검증 DOM 반영
- **위치**: 라인 820–827, 1089–1091, 1113
- **문제**: `handleMatchMessage` 내부에서 서버로부터 수신한 문자열(`msg.opponent.name`, `msg.message` 등)을 그대로 `textContent`에 대입합니다. 현재는 `textContent` 할당이므로 HTML 인젝션(XSS)은 발생하지 않습니다. 그러나 **`msg.opponent.name`에 대한 길이/문자 제한이 전혀 없습니다.** 악의적인 서버가 매우 긴 문자열을 전송하면 레이아웃이 파괴되며, 동일 오리진 상황(또는 프록시 변조)에서 추후 `innerHTML` 기반 코드가 추가될 경우 XSS로 전환될 수 있습니다.

  ```js
  // 라인 820
  let line = netClient.opponent.name || netClient.opponent.id || '-';
  // name, id에 대한 길이·문자 검증 없음
  els.opponent.textContent = line;

  // 라인 1113
  setMatchStatusText(String(msg.message || 'Server error'), 'bad');
  // msg.message 길이 무제한
  ```

- **수정 제안**:
  - `sanitizeName()` (이미 라인 862에 존재)을 `netClient.opponent.name` 할당 시점(라인 1030, 1090, 1109)에도 적용하세요.
  - 서버 오류 메시지도 길이를 제한하세요 (예: `.slice(0, 120)`).

---

### C-2. `statusTone`을 className에 직접 삽입 — CSS 클래스 인젝션 가능성
- **위치**: 라인 817
  ```js
  els.status.className = 'match-meta-val' + (netClient.statusTone ? ' ' + netClient.statusTone : '');
  ```
- **문제**: `netClient.statusTone`은 내부 코드에서만 설정되므로 현재는 안전합니다. 그러나 향후 서버 메시지에서 `tone` 값을 직접 수신하는 경로가 추가될 경우, 임의의 클래스명이 DOM에 삽입됩니다. 화이트리스트 검증이 없습니다.
- **수정 제안**: `tone` 값을 허용 목록(`['ok', 'warn', 'bad', 'strong', '']`)으로 검증하는 헬퍼를 두세요.

---

## [WARNING] 중요 개선 사항

### W-1. 자동재연결 — 재연결 시 새 WebSocket에 이벤트 리스너가 누적될 수 있음
- **위치**: 라인 957–983 (`ws.addEventListener('close', ...)`)
- **문제**: `close` 핸들러 안에서 `connectMatchSocket()`을 재귀 호출하면 **새 WebSocket 인스턴스**에 새 이벤트 리스너 4개(`open`, `message`, `error`, `close`)가 등록됩니다. 재연결 횟수를 최대 1회(`_reconnectAttempts < 1`)로 제한하고 있어 현재는 리스너가 최대 2쌍(초기 + 재연결 1회)만 생성됩니다. 그러나 재연결 제한을 늘릴 경우, 또는 타이밍상 `_manualDisconnect` 플래그가 경쟁 조건으로 잘못 읽힐 경우 리스너 누적이 발생합니다.

  ```js
  // 라인 967
  if (!wasManual && netClient.serverUrl && netClient._reconnectAttempts < 1) {
    netClient._reconnectAttempts++;
    // ...
    netClient._reconnectTimer = setTimeout(function() {
      netClient._reconnectTimer = null;
      if (!netClient.connected && !netClient.connecting) {
        connectMatchSocket(wasQueued || hadMatch); // <-- 새 WS 생성, 새 리스너 4개 추가
      }
    }, 2000);
  ```

- **수정 제안**: 재연결 횟수를 늘릴 계획이 있다면 각 이벤트 리스너를 **named function**으로 추출하고, 소켓 생성 시 클로저 내 변수로 묶어 해당 소켓 인스턴스에만 귀속시키는 방식을 권장합니다. 현재 1회 제한이라면 동작은 안전하지만, 주석으로 이 제약을 명시해두는 것이 좋습니다.

---

### W-2. `closeMatchSocket` 에서 `_manualDisconnect = true` 설정 후 `ws.close()` 비동기성
- **위치**: 라인 889–905
  ```js
  function closeMatchSocket(manualReason) {
    netClient._manualDisconnect = true;
    // ...
    try { netClient.ws.close(); } catch (_) {}
    netClient.ws = null;         // <-- 즉시 null로 설정
    netClient.connected = false;
    // ...
  }
  ```
- **문제**: `ws.close()` 호출 후 `close` 이벤트는 **비동기**로 발생합니다. 그 사이에 `netClient.ws = null`을 동기로 설정하기 때문에 `close` 핸들러(라인 957)에서 `netClient.ws`는 이미 `null`입니다. 핸들러가 `netClient.ws`를 참조하지 않는 현재 코드에서는 문제가 없지만, `_manualDisconnect`를 `false`로 복원하는 로직(라인 961)이 `close` 이벤트 핸들러 안에 있습니다. 만약 `closeMatchSocket()` 직후에 `connectMatchSocket()`을 호출한다면 `close` 이벤트가 **새 소켓의 수명 중에** 도착해 `_manualDisconnect`를 `false`로 덮어쓸 위험이 있습니다.

- **수정 제안**: `_manualDisconnect` 플래그를 `close` 이벤트에서 복원하는 대신, 소켓 인스턴스별 로컬 변수로 관리하거나 `closeMatchSocket` 내부에서 이미 처리된 소켓의 `close` 이벤트를 무시하도록 처리하세요.

---

### W-3. `requestQueueJoin` 에서 `lastResult` 리셋 — 재연결 후 기존 결과 소실
- **위치**: 라인 991
  ```js
  function requestQueueJoin() {
    if (!netClient.connected) {
      connectMatchSocket(true);
      return;
    }
    if (netClient.inMatch) return;
    netClient.lastResult = null;  // <-- 매칭 요청 시 즉시 결과 지움
  ```
- **문제**: REMATCH 버튼 클릭 시 `requestQueueJoin()`이 호출되고, 이때 `lastResult`가 `null`로 설정됩니다. 이것은 의도된 동작이지만, `updateMatchUi()`에서 `showRematch` 조건(`netClient.lastResult !== null`)과 결합되어 버튼이 즉시 사라집니다. 사용자가 매칭을 취소(`queue_leave`)하면 `lastResult`는 `null` 상태이므로 REMATCH 버튼이 다시 나타나지 않습니다. 즉, 한 번 매칭 시도 후 취소하면 이전 게임 결과로 돌아갈 방법이 없습니다.
- **수정 제안**: 매칭 요청 시 `lastResult`를 지우는 시점을 `match_found` 메시지 수신 시점(이미 라인 1033에 있음)으로만 한정하고, `requestQueueJoin`에서는 제거하는 것을 검토하세요. 또는 별도의 `pendingRematch` 플래그로 REMATCH 버튼 표시 여부를 제어하세요.

---

### W-4. `incomingAttackQueue`가 멀티플레이 중단 후에도 초기화되지 않음
- **위치**: 라인 170, `clearMatchSessionState` (라인 878–887)
  ```js
  function clearMatchSessionState() {
    netClient.queued = false;
    netClient.inMatch = false;
    netClient.roomId = null;
    netClient.matchSeed = null;
    netClient.opponent = null;
    netClient.sentGameOverForRoomId = null;
    netClient.lastRemoteState = null;
    if (gameMode === 'versus') setGameMode('solo');
    // incomingAttackQueue 미초기화!
  }
  ```
- **문제**: 매치가 종료되거나 연결이 끊길 때 `incomingAttackQueue`가 초기화되지 않습니다. 이전 매치에서 큐에 쌓인 공격 패킷이 다음 솔로 게임 또는 다음 매치 시작 시 `consumeIncomingAttackCols()`(라인 627)에서 소비되어 엉뚱한 쓰레기 블록이 생성될 수 있습니다.
- **수정 제안**: `clearMatchSessionState()` 또는 `startGame()` 안에서 `incomingAttackQueue = []` 초기화를 추가하세요.

---

### W-5. `opponent_attack` 메시지 역직렬화 — packet 내용 무검증
- **위치**: 라인 1084–1086
  ```js
  case 'opponent_attack':
    if (msg.packet) queueIncomingAttack(Object.assign({}, msg.packet, { source: 'remote' }));
    break;
  ```
- **문제**: `normalizeIncomingAttack()`(라인 601)이 `power`와 `cols`에 대한 경계 처리를 하지만, `cols` 배열의 **길이** 검증이 없습니다. 악의적인 서버가 수천 개의 컬럼 인덱스를 전송하면 `warningJunkCols`가 비정상적으로 커집니다.
  ```js
  // 라인 609
  cols: Array.isArray(p.cols) ? p.cols.map(...) : null,
  // 배열 길이 제한 없음
  ```
- **수정 제안**: `p.cols.slice(0, COLS * 4)` 정도로 길이를 제한하세요.

---

### W-6. `gameLoop`가 `requestAnimationFrame`으로 무한 실행 — `startGame` 재호출 시 중복 루프
- **위치**: 라인 1672, 1798
  ```js
  function gameLoop(ts) {
    // ...
    requestAnimationFrame(gameLoop); // 항상 다음 프레임 요청
  }
  // ...
  startGame();
  requestAnimationFrame(gameLoop); // 최초 1회 시작
  ```
- **문제**: 초기화 시점에 `requestAnimationFrame(gameLoop)`가 1회 호출되고, 이후 `gameLoop` 안에서 매 프레임 재호출합니다. `startGame()`은 게임 상태만 리셋할 뿐 새 루프를 시작하지 않으므로 루프 중복은 발생하지 않습니다. **그러나** 만약 외부 코드가 실수로 `requestAnimationFrame(gameLoop)`를 추가로 호출하면 루프가 2개가 되고 렌더링이 2배 속도로 실행됩니다. 현재 루프 실행 여부를 추적하는 플래그가 없습니다.
- **수정 제안**: `let gameLoopRunning = false` 플래그를 두어 중복 시작을 방어하거나, `gameLoop` 함수 시그니처를 변경해 취소 가능한 핸들을 반환하도록 하세요.

---

## [INFO/IMPROVEMENT] 개선 사항

### I-1. `localStorage` 사용 — 서버 URL이 평문으로 저장
- **위치**: 라인 761–763
  ```js
  if (netClient.serverUrl) localStorage.setItem(STORAGE_KEYS.matchServerUrl, netClient.serverUrl);
  localStorage.setItem(STORAGE_KEYS.matchPlayerName, netClient.playerName);
  ```
- **내용**: WebSocket URL이 `localStorage`에 평문으로 저장됩니다. 동일 오리진의 다른 스크립트(광고, 서드파티 위젯 등)가 삽입된 환경에서 읽힐 수 있습니다. 기능상 문제는 아니나 공유 호스팅 환경에서는 고려할 필요가 있습니다.
- **수정 제안**: 현재 구조(단순 게임 클라이언트)에서는 허용 가능한 수준이지만, `?ws=` 쿼리 파라미터가 우선순위를 가지도록 이미 처리되어 있으므로 별도 조치는 낮은 우선순위입니다.

---

### I-2. `detectInitialLocale`이 `localStorage` 값을 `resolveLocale` 없이 반환
- **위치**: 라인 549–555
  ```js
  function detectInitialLocale() {
    try {
      const saved = localStorage.getItem(STORAGE_KEYS.locale);
      if (saved) return saved;  // <-- 반환 즉시 사용, 검증 없음
    } catch (_) {}
    return (navigator.languages && navigator.languages[0]) || navigator.language || 'en';
  }
  ```
- **문제**: `localStorage`에 임의로 변조된 로케일 문자열(예: `"__proto__"`, `"constructor"`)이 저장되어 있다면 `resolveLocale` 없이 `setLocale`에 전달됩니다. 다행히 `setLocale`이 내부적으로 `resolveLocale()`을 호출하므로 실제 영향은 없습니다. 그러나 코드 의도를 명확히 하기 위해 반환 전에 `resolveLocale`을 적용하는 것이 좋습니다.

---

### I-3. `REMATCH 버튼` 표시 조건 — 게임 오버 오버레이와의 동기화 불일치
- **위치**: 라인 854–859 (`updateMatchUi`), 라인 1119 (`game-over-overlay` HTML)
  ```js
  const showRematch = netClient.connected && !netClient.inMatch && !netClient.queued && netClient.lastResult !== null;
  rematchBtn.style.display = showRematch ? 'block' : 'none';
  ```
- **문제**: REMATCH 버튼은 `game-over-overlay` 내부에 있으므로 게임 오버 오버레이가 보이지 않을 때(솔로 플레이 시작 직후)도 `showRematch`가 `true`이면 버튼이 활성화됩니다. 오버레이는 `display:none` 상태이므로 사용자에게는 보이지 않지만, 상태 자체는 존재합니다. `triggerGameOver()`가 호출되지 않은 상태에서도 버튼이 클릭 가능합니다 (JavaScript로 직접 접근 시).
- **수정 제안**: 실제 UX에는 영향이 없습니다. 다만 `showRematch` 조건에 `gameOver` 플래그를 추가하면 의미가 더 명확해집니다:
  ```js
  const showRematch = gameOver && netClient.connected && !netClient.inMatch && !netClient.queued && netClient.lastResult !== null;
  ```

---

### I-4. `initMatchPanel` 의 rematch 버튼 — 중복 리스너 등록 방어 부재
- **위치**: 라인 1146–1151
  ```js
  const rematchBtn = document.getElementById('rematch-btn');
  if (rematchBtn) {
    rematchBtn.addEventListener('click', function() {
      requestQueueJoin();
    });
  }
  ```
- **문제**: `initMatchPanel()`이 한 번만 호출되므로 (라인 1794) 실제로 중복 등록은 발생하지 않습니다. 그러나 `restart-btn`의 리스너(라인 1754)는 `initMatchPanel` 외부에 등록되는 데 반해, `rematch-btn`의 리스너는 내부에 등록됩니다. 패턴이 일치하지 않아 유지보수 시 혼란을 줄 수 있습니다.
- **수정 제안**: 모든 버튼 리스너를 `initMatchPanel` 안에서 등록하거나, 모두 외부로 꺼내는 방식으로 패턴을 통일하세요.

---

### I-5. `getMatchEls()` — `rematch-btn`이 반환 객체에 포함되지 않음
- **위치**: 라인 725–737
  ```js
  function getMatchEls() {
    return {
      serverUrl: document.getElementById('match-server-url'),
      // ...
      restartBtn: document.getElementById('restart-btn')
      // rematchBtn 없음
    };
  }
  ```
- **문제**: `updateMatchUi()`에서 `rematch-btn`을 직접 `document.getElementById('rematch-btn')`으로 조회합니다(라인 854). 다른 모든 요소는 `getMatchEls()`를 통해 캐싱 없이 매 호출마다 재조회하므로 일관성은 있습니다. 그러나 `rematch-btn`만 예외적으로 직접 조회하여 패턴 불일치가 생깁니다.
- **수정 제안**: `getMatchEls()`에 `rematchBtn: document.getElementById('rematch-btn')`을 추가하세요.

---

### I-6. `ko` / `ja` / `es` 로케일 번역 키 일관성 검토
- **위치**: 라인 262–417

| 키 | `ko` | `ja` | `es` | 비고 |
|---|---|---|---|---|
| `net.rematch` | `'다시 매칭'` | `'再マッチ'` | `'REVANCHA'` | 정상 |
| `net.reconnecting` | `'재연결 중...'` | `'再接続中...'` | `'Reconectando...'` | 정상 |
| `net.socketClosed` | `'연결 끊김'` | `'接続が切れました'` | `'Desconectado'` | 정상 |
| `net.readyPrompt` | 정상 | 정상 | 정상 | |
| `net.needServerUrl` | 정상 | 정상 | 정상 | |
| `fx.merge` | `'{count}개 합성 +{points}'` | `'{count} MERGE +{points}'` (영문 혼용) | `'{count} MERGE +{points}'` (영문 혼용) | **ja/es가 영문 유지** |

- **문제**: `ja`와 `es` 로케일의 `fx.merge` 값이 영문(`MERGE`)을 그대로 사용합니다. 기능 문제는 아니며 의도적인 것일 수 있으나, 다른 `fx.*` 키들이 각 언어로 번역된 것과 일관성이 없습니다.

---

### I-7. `opponent_state` 처리 시 상대방 상태 검증 없이 저장
- **위치**: 라인 1088–1091
  ```js
  case 'opponent_state':
    netClient.lastRemoteState = msg.state || null;
    if (msg.opponent) netClient.opponent = msg.opponent;
    updateMatchUi();
    break;
  ```
- **문제**: `msg.state`에 포함된 `score`, `turn` 값에 대한 타입 검증이 없습니다. `updateMatchUi`에서 `formatInt(netClient.lastRemoteState.score || 0)` 호출 시 `score`가 객체나 배열이면 `Math.floor()`에서 `NaN`이 될 수 있습니다. 실제로 `formatInt`는 `Math.max(0, Math.floor(v || 0))`로 방어하고 있어 화면 렌더링은 안전하지만, 서버 신뢰에 의존하는 형태입니다.
- **수정 제안**: `msg.state.score`와 `msg.state.turn`에 대해 `typeof ... === 'number'` 검증을 추가하세요.

---

### I-8. `t()` 함수에서 변수 치환 시 빈 문자열 대체
- **위치**: 라인 490–492
  ```js
  return value.replace(/\{(\w+)\}/g, function(_, name) {
    return vars[name] === undefined ? '' : String(vars[name]);
  });
  ```
- **문제**: 변수 키가 `undefined`이면 빈 문자열로 치환됩니다. `fx.chain` 키에 `{chain}`이 있는데 호출 시 `vars` 없이 호출되면 `"1x CHAIN!"` 대신 `"x CHAIN!"`이 표시될 수 있습니다. 현재 코드의 모든 호출 지점에서 올바른 변수를 전달하고 있으므로 실제 버그는 아니나, 개발 중 오호출을 감지하기 어렵습니다.
- **수정 제안**: 개발 빌드에서 `console.warn`을 추가하거나 `{key}`를 그대로 유지(치환 안 함)하는 방식으로 디버깅 편의성을 높이세요.

---

## Architecture Considerations

1. **단일 HTML 파일 구조**: 전체 코드가 하나의 `index.html`에 집중되어 있습니다. 현재 규모(~1,800줄)에서는 관리 가능하지만, 기능 추가 시 모듈 분리(`netClient.js`, `gameEngine.js`, `i18n.js`)를 고려하는 것이 좋습니다.

2. **`netClient` 전역 객체**: WebSocket 연결 상태, 자동재연결 타이머, 플레이어 정보 등 모든 네트워크 상태가 하나의 전역 객체에 혼재합니다. `_manualDisconnect`, `_reconnectAttempts`, `_reconnectTimer`와 같은 내부 구현 세부사항과 `playerName`, `serverUrl` 같은 설정값이 동일 레벨에 위치해 있어 향후 상태 추적이 어려울 수 있습니다.

3. **자동재연결 최대 1회 제한**: 현재 `_reconnectAttempts < 1` 조건으로 재연결을 최대 1회로 제한합니다. 이는 의도적으로 단순하게 설계된 것으로 보이며, 무한 재연결 루프를 방지하는 적절한 방어책입니다. 다만 UX 측면에서 네트워크 불안정 환경에서는 재연결 실패 시 사용자에게 명확한 "재연결 버튼"을 제공하는 것이 권장됩니다.

---

## Next Steps (우선순위 순)

1. **[C-1] 즉시**: `netClient.opponent.name/id` 할당 시 `sanitizeName()` 적용, 서버 오류 메시지 길이 제한 추가
2. **[W-4] 단기**: `clearMatchSessionState()` 또는 `startGame()` 내 `incomingAttackQueue = []` 초기화 추가
3. **[W-3] 단기**: REMATCH 버튼 표시 로직과 `lastResult` 초기화 시점 재검토
4. **[C-2] 단기**: `statusTone` 화이트리스트 검증 헬퍼 추가
5. **[W-5] 단기**: `opponent_attack` packet의 `cols` 배열 길이 제한 추가
6. **[I-3] 중기**: REMATCH 버튼 표시 조건에 `gameOver` 플래그 추가
7. **[I-5] 중기**: `getMatchEls()`에 `rematchBtn` 추가하여 패턴 일관성 확보
8. **[I-4] 중기**: 버튼 리스너 등록 패턴 통일

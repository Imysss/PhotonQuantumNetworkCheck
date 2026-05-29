# 진단 모드 절차

진단 모드 호출 시 모델이 따라야 할 단계.

## 1. 진단 범위 확인

사용자가 명확한 범위를 안 주면 묻는다. 진단은 코드 읽기 비용이 크므로 무작정 전체 스캔하지 않는다.

질문 예시:
- "어디부터 점검할까요? (1) 매칭 코드만 (2) 인게임 disconnect 핸들링만 (3) 라이프사이클 처리만 (4) 전체"
- "특정 파일을 지정해주시면 그 파일 + 호출 흐름을 따라 점검합니다."

전체 스캔이 필요한 경우만 광범위 탐색. 보통은 사용자가 지정한 영역 + 직접 호출하는 인접 파일만.

## 2. 코드 매핑

각 시나리오 ID에 대해 코드에서 핸들링이 존재하는지 검색.

검색 키워드 (시나리오 → 키워드):

| ID | 시나리오 | 검색 키워드 |
|----|----------|-------------|
| A1, A9 | 동시 거부 race / rematch 회피 | `JoinRandomRoomAsync`, `OnPlayerEnteredRoom`, `OnPlayerPropertiesUpdate`, `lastOpp`, `isInviteRoom` |
| A2 | 매칭 타임아웃 → 봇 | `matchmakingTimer`, `StartBotMatch`, `Random.Range(9` |
| A3 | 방 생성 실패 | `OnCreateRoomFailed`, `GameIdAlreadyExists`, `ErrorCode` |
| A4 | 방 입장 실패 | `OnJoinRoomFailed`, `GameDoesNotExist`, `GameClosed`, `GameFull` |
| A5 | 방 조건 / 잠금 토스트 | `CustomRoomPropertiesForLobby`, `joinOk`, 잠금 토스트 |
| A6, B9, C5, E3 | 백그라운드 처리 | `OnApplicationPause`, `KeepAliveInBackground`, `backgroundEntered` |
| A7 | 강제 종료 | `PlayerTtl`, ready 핸드셰이크 |
| A8, E4 | 인터넷 끊김 | `Application.internetReachability`, `NetworkReachability` |
| A10 | 매칭 race | `ClientState.Disconnected`, CancellationToken |
| A11 | 동시 매칭 사후 복구 | leave 후 재시도, 2~4초 대기, 1회 한도 |
| A12 | 상태 리셋 | `ResetMatchState`, 메모리/플래그 초기화 |
| A13 | 튜토리얼/에디터 분기 | `Application.isEditor`, 튜토리얼 미완료 |
| B1~B3 | progressive timeout / ready 동기화 | `sceneReady`, `uiReady`, `dataReady`, `gameReady`, 단계별 timeout |
| B4 | 세션 시작 실패 | `SessionRunner.StartAsync`, `ConnectionServiceScope` |
| B5 | RuntimeConfig | `RuntimeConfig`, `BuildRuntimeConfig` |
| B6 | AddPlayer confirm | `CallbackLocalPlayerAddConfirmed`, `GetLocalPlayers` |
| B7 | 초기 데이터 ACK | `EventOn*Applied`, `INITIAL_DATA_TIMEOUT` |
| B8 | 봇 데이터 fetch | `BotDataProvider`, 봇 API 호출 |
| B10 | 프로필 무한 대기 | 프로필 timeout, `Player.CustomProperties` 초기 fetch |
| B11 | disconnect race | `_isShuttingDown` 가드, `QuantumCallback.UnsubscribeListener` |
| C1, C8 | 봇 활성화 | `ActivateBot`, `DeactivateBot`, `OnPlayerLeftRoom`, `BotActivate` |
| C2, C3 | 자신 끊김 | `OnDisconnected`, `DisconnectCause`, `ReconnectAndRejoin` |
| C4 | Desync | `CallbackChecksumErrorFrameDump` |
| C7 | 망 전환 | `ConnectionProtocol`, UDP 설정 |
| C9 | Command 송신 | `SendCommand`, runner null 검증 |
| C10 | 동시 disconnect | 봇 활성화 책임 분리 |
| C11 | 봇 게임 Photon 끊기 | Local 모드 진입 시 `DisconnectAsync`, client 참조 해제 |
| C12 | 재접속 UI 재동기화 | 로컬 UI 강제 갱신, 슬롯별 카메라/진영 분기 |
| D4 | 종료 사유 분류 | `fail_reason` enum, `LifeDepleted`/`ReconnectTimeout` 등 |
| E1, E6 | iOS 인터럽트 | `OnApplicationFocus` |
| E2 | OOM | PlayerPrefs 상태 저장 |
| E5 | 재실행 | 세션 ID 기반 결과 조회 |
| F1 | RuntimeConfig 불일치 | RuntimeConfig 빌드 위치, broadcast 로직 |
| F2 | 버전 차이 | 앱 버전 교환, 강제 업데이트 |
| F3 | Late join | `Room.IsOpen`, snapshot |
| F4 | Frame catch-up | `RollbackWindow`, `SessionConfig` |
| F5 | Pause | `runner.Session.Pause()` |
| G1 | 사전 검사 | 에너지 부족, 버전 검사, 시즌 데이터 |
| G2 | 자원 차감 시점 | 차감 commit point, 환불 API |
| G3 | sanitization | `SanitizeNickname`, 제어 문자 제거 |
| G4 | 매칭 mutex | `_isStartingMatch`, `interactable = false` |
| H1 | 재접속 오버레이 UI | `OnLocalDisconnected` 구독자, `UI_ReconnectOverlay` |
| H2~H6 | UX gap | 발화-구독 매칭, 진행 표시 유무 |

각 키워드 검색 → 결과 있으면 "구현 흔적 있음" → 핸들러 내용을 읽어 완전성 판정.

## 3. 상태 판정 기준

| 상태 | 기준 |
|------|------|
| **OK** | 핸들러 존재 + 시나리오의 모든 권장 조치가 실제 코드에 반영됨 |
| **부분** | 핸들러 존재하지만 일부 권장 조치 누락 (예: timeout 있지만 fallback 없음) |
| **미구현** | 핸들러 자체가 없거나 빈 메서드 |
| **불확실** | 코드를 다 읽지 못했거나 흐름이 복잡해서 단정하기 어려움 |

추측 금지. 불확실하면 "불확실"로 표시하고 사용자에게 추가 확인을 요청.

## 4. 심각도 판정

reference 파일에 명시된 우선순위를 1차 기준:
- 高 → 심각도 高
- 中 → 심각도 中
- 低 → 심각도 低

단, 프로젝트 상황에 따라 조정:
- 출시 임박 PR이면 中 → 高으로 격상.
- 코드가 활발히 바뀌는 영역이면 잠재 회귀 위험 → 격상.

## 5. 표 출력

[SKILL.md](../SKILL.md)의 출력 형식 그대로:

```
| ID | 시나리오 | 상태 | 심각도 | 근거 (파일:라인) | 권장 조치 |
```

정렬: 심각도 高 → 中 → 低. 같은 심각도 내에선 `미구현` → `부분` → `불확실` → `OK` 순.

## 6. 권장 조치 섹션

표 아래에 "권장 조치" 섹션. 미구현/부분 항목 위주로:

- 무엇을 해야 하는지 한 문장 요약.
- 관련 reference 파일 링크.
- 가능하면 권장 코드 패턴 1~2줄.

## 7. 다음 단계

표 + 권장 조치 마지막에 "다음 단계" 한 줄. 사용자가 어디부터 손대면 좋을지.

예시:
> 다음 단계: C2 (자신 끊김 핸들링) 미구현 — 가장 빈도 높은 케이스이므로 먼저 처리 권장.

## 8. 진단 결과 마무리 — 후속 행동은 별도 승인

진단 결과를 보고한 뒤 코드 수정을 자동으로 진행하지 않는다. "이 항목 고쳐드릴까요?" 같은 질문으로 사용자의 명시 승인을 받고 다음 동작을 결정.

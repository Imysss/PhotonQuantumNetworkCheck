# 매칭 기능 PR 체크리스트

매칭 코드를 추가/수정할 때 PR 머지 전 점검.

각 항목 옆 ID는 [references/A-matchmaking.md](../references/A-matchmaking.md) 시나리오 참조.

## 랜덤 매칭

- [ ] **A1** 동시 거부 race: `OnPlayerEnteredRoom` + `OnPlayerPropertiesUpdate` 양쪽에서 거부 판정.
- [ ] **A1** Tie-break: 양쪽이 동시에 거부 의도일 때 한쪽만 leave하는 결정적 규칙 (UserId/ActorNumber 비교).
- [ ] **A2** 매칭 타임아웃 상한 설정 (권장 9~13초, 랜덤화).
- [ ] **A2** 봇 fallback 진입 시 방 `IsOpen = false`, `IsVisible = false`.
- [ ] **A2** 봇 fallback UI 명시 ("AI 대전").

## 방 코드 매칭

- [ ] **A3** `OnCreateRoomFailed`에서 `GameIdAlreadyExists` 시 자동 재시도 (3~5회).
- [ ] **A3** 방 코드 길이 검토 (4자리는 충돌 빈도 높음).
- [ ] **A4** 입장 실패 에러 코드별 사용자 메시지 분기 (`GameDoesNotExist`, `GameClosed`, `GameFull`).
- [ ] **A4** 입력 코드 trim + 대소문자 정규화.
- [ ] **A5** 방 조건 (난이도/스테이지)을 `CustomRoomPropertiesForLobby`에 노출.
- [ ] **A5** 입장 시도 직전 클라이언트 검증 → 실패 시 입장 자체를 막음.

## 백그라운드 / 종료

- [ ] **A6** `OnApplicationPause(true)` + 페이즈가 Lobby면 즉시 `DisconnectAsync()`.
- [ ] **A7** 양쪽 매칭 직후 ready 핸드셰이크 (5~10초 timeout).
- [ ] **A7** `Room.PlayerTtl = 0` 설정 (즉시 `OnPlayerLeftRoom` 발화).

## 네트워크 끊김

- [ ] **A8** 매칭 중 `Application.internetReachability` 폴링 (2초 간격).
- [ ] **A8** `OnDisconnected(DisconnectCause cause)` 핸들러에서 cause별 분기 (`ClientTimeout`, `ServerTimeout`, `DisconnectByClientLogic` 등).
- [ ] **A8** 끊김 후 client 상태 클린업 (재인스턴스 or `ReconnectAndRejoin`).

## 매칭 상태 가드

- [ ] **A9** 직전 상대 거부 키는 양방향 비교 (`mineLastOpp == currentUserId` OR `currentLastOpp == myUserId`).
- [ ] **A9** 랜덤 vs 초대 구분 — `isInviteRoom` 플래그 셋팅 시 거부 판정 / 기록 skip.
- [ ] **A9** AI 판 직후엔 직전 상대 ID 삭제 (봇 ID로 사람 매칭 막히지 않게).
- [ ] **A10** 매칭 시작 함수에 상태 가드:
  ```csharp
  if (_client.State != ClientState.Disconnected &&
      _client.State != ClientState.PeerCreated) {
      await _client.DisconnectAsync();
  }
  ```
- [ ] **A10** 취소 버튼은 누른 직후 비활성화 → `DisconnectAsync` 완료 후 활성화.
- [ ] **A10** CancellationToken으로 매칭 작업 안전 취소.

## 동시 매칭 / 메모리 / 분기

- [ ] **A11** 방 생성 후 2~4초 대기 → 1회 leave + JoinRandomRoom 재시도 (사후 복구).
- [ ] **A11** 사후 복구는 최대 1회 — 무한 retry 금지.
- [ ] **A12** 매칭 시작 첫 줄에 `ResetMatchState()` — 타이머/플래그/캐시 전부 초기화.
- [ ] **A12** 직전 상대 ID는 유지 (AI 판 직후만 삭제).
- [ ] **A13** 튜토리얼 미완료 → 실시간 서버 없이 즉시 봇.
- [ ] **A13** Editor 환경 → 즉시 봇 (`forceMultiplayerInEditor`로 override 가능).

## 일반

- [ ] 모든 매칭 API 호출에 try-catch + 로깅.
- [ ] 매칭 진입 / 성공 / 실패 / 취소 분기마다 텔레메트리.
- [ ] 매칭 화면에서 사용자가 무엇을 기다리는지 명확히 표시 (단계별 진행률).
- [ ] G 카테고리 사전 검사 통과 후에만 매칭 시작 호출 (G1 참조).

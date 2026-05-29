# 인게임 네트워크 처리 체크리스트

Quantum 세션이 시작된 후의 동작을 점검. ID는 [references/C-in-game-disconnect.md](../references/C-in-game-disconnect.md) / [references/F-quantum-determinism.md](../references/F-quantum-determinism.md) 참조.

## 상대 이탈 / 봇 활성화

- [ ] **C1** `OnPlayerLeftRoom` 핸들러에서 현재 페이즈 분기 (Lobby/Result는 skip).
- [ ] **C1** 봇 활성화 시 방 `IsOpen = false`, `IsVisible = false`.
- [ ] **C1** 봇 PlayerRef 계산 (로컬의 반대 슬롯).
- [ ] **C1** 봇 활성화는 deterministic command로 시뮬에 주입 (`BotActivateCommand` 등).
- [ ] **C1** UI에 "상대가 이탈해 AI가 대신합니다" 알림.
- [ ] **C8** 봇 활성화 후 원래 상대 복귀 정책 결정 (허용 vs 차단). 일관 적용.

## 자신 끊김 / 재접속

- [ ] **C2** `OnDisconnected` 핸들러 — 페이즈가 Gameplay면 로컬 재접속 타이머 시작 (30s).
- [ ] **C2** 재접속 중 UI 오버레이 ("재연결 시도 중... XX초 남음").
- [ ] **C2** `Application.internetReachability != NotReachable` 일 때 `ReconnectAndRejoin()`.
- [ ] **C2** 타이머 만료 시 결과 화면 전환.
- [ ] **C3** `DisconnectCause`별 분기 (`ClientTimeout`, `ServerTimeout`, `MaxCcuReached`, `InvalidAuthentication` 등).

## 백그라운드

- [ ] **C5** `OnApplicationPause(true)` 진입 시각 기록.
- [ ] **C5** `OnApplicationPause(false)` 복귀 시 elapsed 계산.
- [ ] **C5** `elapsed >= 30s` 면 로컬 재접속 실패 처리 → 결과 화면.
- [ ] **C6** `elapsed < 30s` 면 Quantum catch-up 대기 (별도 처리 없음).
- [ ] **C6** catch-up 중 input lock + UI "동기화 중" 표시.
- [ ] Photon `Handler.KeepAliveInBackground = 30000` 과 로컬 타이머 값 동일.

## 망 전환

- [ ] **C7** UDP 프로토콜 사용 (`AppSettings.Protocol = ConnectionProtocol.Udp`).
- [ ] **C7** `NetworkReachability` 변화 텔레메트리.

## Command 송신

- [ ] **C9** Command sender는 최신 `QuantumRunner` 참조 보관.
- [ ] **C9** 송신 직전 `runner != null && runner.IsRunning` 검증.
- [ ] **C9** 봇 게임 전환 시 새 runner 참조 등록.

## 결정론

- [ ] **F1** RuntimeConfig는 한쪽이 빌드 → Photon RoomProperty로 broadcast (양쪽 독립 빌드 X).
- [ ] **F1** `RuntimeConfig.Seed` 명시 설정 (랜덤 결과 양쪽 동일).
- [ ] **F2** 매칭 시 양쪽 앱 버전 교환 → 다르면 매칭 거부.
- [ ] **F2** 강제 업데이트 시스템 (서버에서 최소 버전).
- [ ] **F2** AssetGuid + content hash 시작 시 비교.
- [ ] **F3** Late join 차단 정책 (또는 snapshot 처리).
- [ ] **F4** `SessionConfig.RollbackWindow` 모바일용으로 확대 (90~120).
- [ ] **F4** 격차 큰 경우 input lock + UI 표시.
- [ ] **F5** 멀티 모드에서 `runner.Session.Pause()` 호출 금지.

## Checksum / Desync

- [ ] **C4** `CallbackChecksumErrorFrameDump` 구독.
- [ ] **C4** 발생 시 frame dump 저장 + 결과 화면 전환.
- [ ] **C4** 시뮬레이션 코드(Quantum systems)에서 비결정성 원천 제거:
  - [ ] `UnityEngine.Random` → `Frame.RNG`
  - [ ] `Time.deltaTime` → `Frame.DeltaTime`
  - [ ] `DateTime.Now` → `Frame.Number`로 시간 추론
  - [ ] Dictionary 순회 시 정렬된 키 사용
- [ ] **C4** Checksum 에러 빈도 텔레메트리.

## 동시 disconnect / 봇 race

- [ ] **C10** 봇 활성화는 **이탈하지 않은 쪽**만 trigger.
- [ ] **C10** 봇 활성화 command frame 기록 → catch-up 후에도 같은 frame에서 활성화.

## 봇 게임 / 재접속 후 UI

- [ ] **C11** 봇 게임(Quantum Local 모드) 진입 시 Photon `DisconnectAsync()` + client 참조 해제.
- [ ] **C11** 봇 게임 중 어떤 Photon 콜백도 발화 안 함.
- [ ] **C12** 재접속 성공 시 로컬 UI 강제 재동기화 (점수/체력/자원/슬롯별 카메라·진영 표시 등).
- [ ] **C12** 재동기화 직후 1~2 frame 입력 잠금.

## 텔레메트리 사유 분류

- [ ] **D4** 종료 사유 enum 사용 (`GameConditionLost`/`ReconnectTimeout`/`OpponentLeft`/`ChecksumError` 등).
- [ ] **D4** 대체 세션 진입 경로 분류 (`ReplacementFromStart`/`ReplacementMidMatch`/`ReplacementMatchFailed`).
- [ ] **D4** 결과 팝업 표시 후 사유 플래그 리셋.

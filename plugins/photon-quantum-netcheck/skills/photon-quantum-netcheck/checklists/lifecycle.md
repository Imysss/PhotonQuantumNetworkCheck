# 앱 라이프사이클 체크리스트

OS 인터럽트 / 백그라운드 / 종료 처리를 점검. ID는 [references/E-mobile-lifecycle.md](../references/E-mobile-lifecycle.md) 참조.

## Unity 라이프사이클 핸들러

- [ ] `OnApplicationPause(bool pause)` 구현 — 실제 백그라운드 진입/복귀.
- [ ] `OnApplicationFocus(bool focus)` 구현 — transient 인터럽트 (iOS 전화 등).
- [ ] iOS의 transient interrupt는 focus만 발화 → 게임 일시정지 X (**E1**).
- [ ] `OnApplicationQuit`에 의존하지 말 것 (실행 보장 없음, **E5**).

## 페이즈별 동작 분기

라이프사이클 핸들러는 항상 현재 페이즈 (Lobby / SceneLoading / Gameplay / Result)를 보고 분기:

- [ ] **Lobby + pause=true** → 즉시 `DisconnectAsync()` (A6).
- [ ] **SceneLoading + pause=true** → 백그라운드 진입 시각 기록 (B9).
- [ ] **Gameplay + pause=true** → 백그라운드 진입 시각 기록 (C5, E3).
- [ ] **Result** → 별도 처리 없음 (보상 처리 중이면 완료 대기).

## 복귀 처리

- [ ] **C5/B9** 복귀 시 elapsed 계산.
- [ ] elapsed >= 30s → 로컬 재접속 실패 → 결과 화면.
- [ ] elapsed < 30s → Quantum catch-up 대기.
- [ ] Photon `Handler.KeepAliveInBackground = 30000` 동일 값.

## 네트워크 폴링

- [ ] **A8/E4** 매 2초 `Application.internetReachability` 체크.
- [ ] `NotReachable` 감지 즉시 UI 알림.
- [ ] 회복 시 자동 `ReconnectAndRejoin()` 시도.

## 메모리 / 강제 종료

- [ ] **E2** 백그라운드 진입 시 현재 게임 상태를 PlayerPrefs/캐시에 저장.
- [ ] **E2** 백그라운드 직전 대용량 asset 해제 (OOM kill 방지).
- [ ] **E5** 재실행 시 이전 세션 ID로 서버에서 결과 조회.
- [ ] **E5** 강제 종료 시점 미완료 보상은 재실행 시 클레임 큐로.

## 화면 잠금 / 알림

- [ ] **E3** 화면 잠금은 `OnApplicationPause(true)`로 처리.
- [ ] **E6** 시스템 알림으로 인한 포커스 변화도 동일 처리.
- [ ] **E6** 중요 시점(보스전 등)엔 알림 차단 권장 UI (선택사항).

## 시뮬레이션 안전

- [ ] **F5** 멀티 모드에선 `runner.Session.Pause()` 호출 금지.
- [ ] 대신 백그라운드 처리를 라이프사이클 핸들러에 집중.
- [ ] 양쪽 동시 일시정지가 필요하면 deterministic command로 시뮬 내부에서 표현.

## 로그 / 진단

- [ ] 라이프사이클 이벤트마다 로그 (시각, 페이즈, 이전 상태).
- [ ] elapsed 시간, internetReachability 변화 함께 기록 → 트러블슈팅 시 추적 가능.

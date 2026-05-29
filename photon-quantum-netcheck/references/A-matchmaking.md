# A. 매칭 단계 (Lobby phase)

매칭 시작 ~ 양쪽 플레이어가 같은 방에 들어와 Quantum 세션이 시작되기 직전까지.

이 페이즈는 Photon Realtime의 `LoadBalancingClient` / `RealtimeClient` 가 주체. 아직 `QuantumRunner`는 시작되지 않은 상태.

---

## A1. 랜덤 매칭 — 동시 거부 race

**발생 조건**
- 두 클라이언트가 거의 동시에 `JoinRandomRoomAsync`를 호출하고 같은 방에 들어옴.
- 양쪽 모두 "직전 상대"가 같음 → 둘 다 서로를 거부해야 함.
- 거부 판정에 사용되는 CustomProperty가 한쪽에만 도달.

**증상**
- 한쪽은 거부하고 나갔는데 다른 쪽은 그대로 머물러 봇 매칭으로 전환되지 않음.
- 또는 양쪽 다 거부해서 매칭 자체가 무한히 실패.

**근본 원인**
- Photon CustomProperty 갱신이 비동기. `SetCustomProperties` 호출 시점과 상대가 `OnPlayerPropertiesUpdate`로 받는 시점에 갭이 있음.
- `OnPlayerEnteredRoom` 시점엔 상대의 latest property가 아직 도착 안 했을 수 있음.

**대응**
- 거부 판정은 **두 곳**에서 실행: `OnPlayerEnteredRoom` 시점 + `OnPlayerPropertiesUpdate` 시점 (해당 키가 변했을 때 fallback).
- 거부 키워드를 양쪽이 동시에 셋팅했을 때 deterministic하게 한쪽만 leave하도록 ActorNumber 또는 UserId 비교로 tie-break.
- 거부 후 재매칭 시도 횟수에 상한(e.g. 3회) → 초과 시 봇 fallback.

**우선순위**: 高

---

## A2. 랜덤 매칭 — 타임아웃 후 봇 fallback

**발생 조건**
- `JoinRandomRoomAsync`가 일정 시간(보통 8~15초) 내에 상대를 못 찾음.
- 또는 방은 만들었으나 두 번째 플레이어가 안 들어옴.

**증상**
- 매칭 화면에 계속 머무름.
- 사용자가 답답해서 취소하거나 앱을 끔.

**근본 원인**
- 한가한 시간대 / 특정 난이도 / 특정 지역 매칭 풀에 인원 부족.
- `TypedLobby` 필터가 너무 엄격해서 매칭 후보가 없음.

**대응**
- 매칭 타이머에 상한값(권장 9~13초, 랜덤화로 동시 fallback 방지) → 초과 시 봇 세션으로 전환.
- 봇 세션 시작 전 방 `IsOpen = false` + `IsVisible = false` 로 변경 → 뒤늦게 들어오는 매칭 후보 차단.
- 봇 fallback 시점에 UI 상 "AI 대전으로 진행" 같은 알림을 명시.

**우선순위**: 高

---

## A3. 방 생성 — 코드 충돌

**발생 조건**
- 방 코드(roomName)를 4~6자리 숫자/문자로 생성.
- 동시에 생성된 방의 코드가 중복.

**증상**
- `CreateRoom` 응답에서 `ErrorCode.GameIdAlreadyExists` (32766).
- 또는 응답이 없고 timeout.

**근본 원인**
- 짧은 코드 공간(4자리 = 10000개) + 활성 방 수가 많을 때 충돌 확률 상승.
- 재시도 로직 없으면 사용자는 "방 생성 실패"로 받아들임.

**대응**
- `OnCreateRoomFailed(short returnCode, string message)` 콜백에서 `returnCode == ErrorCode.GameIdAlreadyExists` 면 새 코드 생성 후 자동 재시도.
- 재시도 횟수 상한 (3~5회) + 초과 시 사용자에게 명시.
- 충돌 빈도가 잦으면 코드 길이를 늘릴 것 (4자리 → 6자리 = 100만개).

**우선순위**: 中

---

## A4. 방 입장 — 코드 오타 / 만료된 방

**발생 조건**
- 사용자가 친구에게 받은 코드로 `JoinRoomAsync` 호출.
- 코드 오타, 또는 방장이 이미 방을 닫고 나간 후.

**증상**
- `OnJoinRoomFailed`에서 `ErrorCode.GameDoesNotExist` (32758) 또는 `GameFull` (32765) 또는 `GameClosed` (32764).
- 사용자가 "왜 안 들어가지" 하고 같은 코드로 계속 시도.

**근본 원인**
- 방 코드는 일회성. 방장이 나간 직후 잠깐의 윈도우 동안 코드가 살아 있다가 사라짐.
- 에러 코드별로 의미가 다른데 UI는 "입장 실패"로 통합 표시하면 진단 불가.

**대응**
- 에러 코드별로 사용자 메시지 분기:
  - `GameDoesNotExist` → "존재하지 않는 방입니다. 코드를 다시 확인해주세요."
  - `GameClosed` → "이미 시작된 방입니다."
  - `GameFull` → "방이 가득 찼습니다."
- 입력 코드는 자동으로 trim + 대소문자 정규화 후 검증.
- 입장 실패 후 3회 같은 코드 시도하면 UI에 안내 추가.

**우선순위**: 中

---

## A5. 방 입장 — 조건 불일치 (난이도/스테이지/잠금)

**발생 조건**
- 방장이 특정 난이도/스테이지로 방을 만듦.
- 입장자의 잠금 해제 상태가 그 조건을 만족하지 않음 (해당 난이도 미해금 등).

**증상**
- 입장은 성공했지만 한쪽이 게임을 시작할 수 없는 상태.
- 게임 시작 시점에 desync 또는 RuntimeConfig 빌드 실패.
- 또는 입장 거부됐는데 사용자에게 사유가 안 보임.

**근본 원인**
- Photon 자체엔 "입장 가능 조건"이 없음. CustomRoomProperties로 표시할 뿐, 검증은 클라이언트 책임.
- 검증을 게임 시작 직전에 하면 이미 매칭이 성립한 상태라 양쪽 다 취소해야 함.

**대응**
- 방 생성 시 `CustomRoomPropertiesForLobby`에 조건 키들을 노출 (`difficulty`, `stage`, `minLevel` 등).
- 입장 시도 직전에 클라이언트가 로비에서 가져온 방 정보를 검증 → 실패 시 입장 자체를 막음.
- 입장 후 검증(예: 방장이 조건을 바꿨을 때)은 CustomPlayerProperty `joinOk = true/false` 송신 → false면 leave.
- **잠금/미해금 사유는 별도 메시지**로 표시 (단순 "입장 실패"가 아니라 사유별 안내). 사용자가 다음 액션을 알 수 있게.
- 게임 시작 데이터(난이도/맵/시작 자원/보상 등)는 **방장 기준**으로 통일 — 입장자의 UI 설정값과 다르더라도 방장 값으로 진행.

**우선순위**: 中

---

## A6. 매칭 중 백그라운드 진입

**발생 조건**
- 사용자가 매칭 대기 화면에서 홈 버튼 누름 / 다른 앱으로 전환.
- 아직 방에 혼자 있거나 상대 진입 직전.

**증상**
- 백그라운드에서 매칭이 성립되고 상대가 들어옴 → 사용자는 모름.
- 30초 후 Photon이 자동으로 끊으면 상대는 "매칭 후 즉시 이탈"로 본인을 인식 → 봇 매칭 시작 → 사용자가 돌아왔을 때 이미 봇 게임 진행 중.

**근본 원인**
- `LoadBalancingClient`는 `Handler.KeepAliveInBackground = 30000` 기본값으로 30초간 백그라운드에서도 연결 유지 시도.
- 매칭 단계는 인게임과 달리 봇 대체 의미가 없음 — 사용자는 매칭 결과를 봐야 함.

**대응**
- `OnApplicationPause(bool pause)` 에서 현재 페이즈가 "Lobby" 면 `pause == true` 시 즉시 `DisconnectAsync()` 호출.
- 매칭 대기 UI를 닫고 "매칭이 취소되었습니다" 알림 표시.
- 인게임 페이즈에서는 별도 처리 (C5, C6 참조).

**우선순위**: 高

---

## A7. 매칭 중 앱 강제 종료 → 유령 방

**발생 조건**
- 매칭이 성립되어 방에 두 명이 들어옴.
- 한쪽이 게임 시작 전에 앱을 스와이프해서 종료.

**증상**
- Photon은 클라이언트가 살아있는 줄 알고 PlayerTtl이 지나기 전엔 방을 유지.
- 다른 한쪽은 "상대 입장 직후 응답 없음" 상태로 멍하니 기다림.

**근본 원인**
- 강제 종료는 OS가 프로세스를 죽이는 거라 클라이언트에서 사전 알림 불가.
- Photon이 inactivity 감지로 disconnect 처리할 때까지 ~10초 (TCP) 또는 ~5초 (UDP).

**대응**
- 매칭 직후 양쪽이 "ready" 신호(CustomPlayerProperty)를 교환하는 핸드셰이크 추가. 한쪽이 일정 시간(5~10초) 내 ready 안 보내면 leave 처리.
- 또는 `Room.PlayerTtl = 0` 으로 설정해서 disconnect 시 즉시 ActorList에서 제거되도록 → `OnPlayerLeftRoom` 빠르게 발화.
- 강제 종료 직전엔 `OnApplicationQuit`이 일부 케이스에서 호출됨 → 가능하면 `Disconnect()` 시도 (하지만 비동기 IO는 못 끝날 가능성 있음).

**우선순위**: 高

---

## A8. 매칭 중 인터넷 끊김

**발생 조건**
- 매칭 대기 중 Wi-Fi 끊김 / LTE 신호 사라짐 / 비행기 모드.

**증상**
- Photon이 자체 timeout (보통 10초)에 도달할 때까지 매칭 화면이 멈춤.
- 그 후 `OnDisconnected(DisconnectCause.ClientTimeout)` 또는 `ServerTimeout` 발화.

**근본 원인**
- `Application.internetReachability`는 즉시 변하지 않음 (OS의 캐시).
- Photon은 자체 ping으로 끊김을 감지하지만 그 사이엔 사용자에게 피드백 없음.

**대응**
- 매칭 중 매 N초(예: 2초)마다 `Application.internetReachability` 체크 → `NetworkReachability.NotReachable` 이면 사용자에게 즉시 알림 + 매칭 취소.
- `OnDisconnected` 콜백을 항상 구현. `DisconnectCause`를 보고 분기:
  - `ClientTimeout` / `ServerTimeout` → 네트워크 끊김 메시지
  - `DisconnectByClientLogic` → 정상 종료 (메시지 표시 안 함)
- 끊김 후 사용자가 다시 매칭을 누르기 전에 `RealtimeClient` 상태를 클린(재인스턴스 또는 `ReconnectAndRejoin` 시도).

**우선순위**: 高

---

## A9. 같은 상대 재매칭 방지 — 랜덤 vs 초대 구분

**발생 조건**
- 직전 매칭의 상대를 다음 매칭에서 다시 만나지 않도록 "직전 상대 UserId"를 비교.
- 두 클라이언트가 동시에 거부 키를 셋팅 → 서로 직전 상대가 다른데도 거부 트리거됨.

**증상**
- 매칭이 성립했다가 양쪽 다 자동으로 leave → 봇 fallback.
- 사용자는 매칭이 두 번 깨진 것처럼 보임.
- 또는 초대 방에서 친구와 재대전하려는데 거부됨.

**근본 원인**
- 양쪽 클라이언트가 같은 키를 본인 기준으로 셋팅하지만 의미가 어긋남.
- "직전 상대 == 현재 상대" 비교 자체는 맞지만, 한쪽만 거부해야 하는 케이스(예: 한쪽은 처음 보는 상대인데 상대 입장에선 직전 상대)를 처리 못 함.
- 초대 방(친구와 재대전 의도)에서도 거부가 발동하면 의도와 어긋남.

**대응**
- 거부 키는 양방향 비교: `myLastOpponent == currentOpponent.UserId` OR `currentOpponent.lastOpponent == myUserId`. 어느 한쪽이라도 true면 거부.
- 거부 책임은 한쪽만 짐 (`UserId` 사전순 비교로 한쪽이 leave). 양쪽이 다 leave하면 봇 fallback이 양쪽에서 발동.
- **랜덤 매칭 vs 초대 방 구분**: 방 생성/입장 경로에서 `isInviteRoom` 플래그 셋팅 → true면 거부 판정 자체를 skip. 직전 상대 ID 기록도 skip.
- **AI 판 직후 직전 상대 ID 처리**: 직전 판이 봇과의 게임이면 다음 매칭 전 ID를 삭제 (봇 ID로 사람 매칭이 막히지 않게). 사람과의 매칭이었으면 ID 유지.
- 거부 시 사용자에게 별도 UI를 띄우지 않고 silent하게 재매칭 시도 (3회 한도) → 그래도 실패면 봇 fallback.
- 거부 검사 시점은 여러 곳: 랜덤 입장 직후 / 상대 진입 / 동시 매칭 사후 복구 / 상대 `lastOpp` 늦은 도착.

**우선순위**: 中

---

## A10. 매칭 취소 후 즉시 재시작 (race)

**발생 조건**
- 사용자가 매칭 대기 중 취소 → 그 직후 다시 매칭 시작.
- `DisconnectAsync()`가 끝나기 전에 `ConnectToRoomAsync()` 호출.

**증상**
- `InvalidOperationException` 또는 `RealtimeClient is already connected/connecting`.
- 클라이언트 상태가 꼬여서 다음 매칭이 영영 시작 안 함.

**근본 원인**
- `RealtimeClient.State`가 `Disconnecting` 단계에 머무는 동안 새 연결 요청을 받음.
- async/await 흐름에서 cancel과 재시작이 중첩.

**대응**
- 매칭 시작 함수에 mutex 또는 상태 가드:
  ```csharp
  if (_client.State != ClientState.Disconnected &&
      _client.State != ClientState.PeerCreated) {
      await _client.DisconnectAsync();
  }
  await _client.ConnectToRoomAsync(args);
  ```
- 취소 버튼은 누른 직후 비활성화 → `DisconnectAsync` await 완료 후 다시 활성화.
- CancellationToken을 사용해서 진행 중인 매칭 작업을 안전하게 취소.

**우선순위**: 中

---

## A11. 동시 매칭 — 각자 방만 생긴 사후 복구

**발생 조건**
- 두 사용자가 거의 동시에 매칭 시작.
- `JoinRandomRoomAsync` 가 양쪽 모두 "조건 맞는 방 없음" 응답을 받고 각자 새 방 생성.
- 결과: 같은 매칭 풀에 1인 방이 두 개.

**증상**
- 두 사용자 모두 무한히 상대를 못 만남.
- 9~13초 후 양쪽 다 봇 fallback (정상 매칭이 가능했는데 둘 다 봇으로).

**근본 원인**
- `JoinRandomOrCreateRoomAsync` 의 원자성 보장은 없음 — 응답 사이에 race window 존재.
- 한 번 방을 만들면 자기 방엔 머무르고 다른 방을 다시 찾지 않음.

**대응**
- 방 생성 후 **2~4초 대기 → 한 번만** 다시 leave + JoinRandomRoom 시도. (대기 시간을 랜덤으로 두면 양쪽이 동시에 leave할 확률 줄어듦.)
- 사후 복구는 **1회만**. 너무 많이 시도하면 매칭 풀이 불안정해짐.
- 사후 복구 실패해도 일반 매칭 타이머는 계속 진행 (별도 봇 트리거 X).
- 사용자에게는 별도 UI 없음 — silent 복구.

**우선순위**: 中

---

## A12. 매칭 시작 직전 — 메모리/플래그 리셋

**발생 조건**
- 새 매칭(랜덤/방 생성/코드 입장) 직전.
- 이전 판의 메모리 상태(매칭 타이머, 봇 활성 플래그, 이탈 플래그, 준비 신호, 상대 프로필, AI 프로필 API 캐시 등)가 남아 있음.

**증상**
- 새 매칭에서 이전 판의 봇 fallback이 즉시 트리거됨.
- 이전 판의 "준비됨" 플래그가 새 판에서도 true로 인식 → 핸드셰이크 skip.
- 이전 판의 늦은 콜백 이벤트가 새 판 상태를 오염.

**근본 원인**
- 매니저가 singleton이고 매칭마다 인스턴스를 새로 만들지 않음.
- async 이벤트 흐름에서 늦게 도착한 콜백이 새 상태에 적용됨.

**대응**
- 매칭 시작 함수 첫 줄에 `ResetMatchState()` 호출 — 다음을 모두 초기화:
  - 매칭 타이머, 봇 활성 플래그, 이탈 플래그
  - 단계별 ready 신호 (sceneReady, uiReady, dataReady, gameReady)
  - 상대 프로필 캐시, AI 프로필 API 캐시
  - opponentDisconnected, backgroundEnteredAt, localReconnectTimer
- 직전 상대 ID는 **유지** (rematch 회피용, A9 참조). AI 판 직후면 삭제.
- 이전 판의 Quantum runner는 **여기서 강제 종료하지 않음** — 게임 씬 언로드에 맡김 (다음 씬 진입 시 자동 cleanup).
- 의도적 disconnect 진행 중 늦게 도착한 이벤트는 무시 플래그로 차단 (B11 참조).

**우선순위**: 高

---

## A13. 튜토리얼 / 에디터 — 실시간 서버 없이 즉시 봇

**발생 조건**
- 멀티 튜토리얼 (아직 사람과 매칭 경험 없는 사용자).
- Unity Editor 환경 (개발자 로컬 테스트).

**증상**
- 실시간 서버 접속을 시도하면 시간만 소비.
- 튜토리얼에서 사용자가 봇과 자연스럽게 학습해야 하는데 9~13초 매칭 대기 발생.

**근본 원인**
- 실시간 서버는 정상 사용 케이스. 튜토리얼/에디터는 환경이 다르므로 분기 필요.
- 에디터에서 Photon 연결 안정성이 낮을 수 있음 (네트워크 권한 등).

**대응**
- 매칭 시작 시점에 환경 검사:
  - 튜토리얼 미완료 → 실시간 서버 없이 즉시 로컬 모드(대체 세션 진입).
  - `Application.isEditor` && 일반 매칭 → 즉시 로컬 모드.
- 로컬 모드에선 `JoinRandomRoomAsync` 호출 안 함 → 동시 매칭 복구 (A11) 도 skip.
- 에디터에서 멀티 디버그가 필요한 케이스를 위해 별도 플래그(`forceMultiplayerInEditor`)로 override 가능하게.
- 로컬 모드 진입 후 `QuantumRunner`는 `DeterministicGameMode.Local`로 시작 (대체 세션 정책 = 봇/AI/무승부 등은 게임 정책에 따라).

**우선순위**: 中

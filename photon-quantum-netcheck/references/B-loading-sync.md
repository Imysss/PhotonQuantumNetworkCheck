# B. 로딩 동기화 (SceneLoading phase)

매칭 성립 → 양쪽이 게임 씬을 로드 → Quantum 세션 시작 → 초기 데이터(플레이어 캐릭터/장비/컴포넌트 등 게임 시작에 필요한 모든 데이터) 동기화까지.

이 페이즈는 Photon Realtime의 CustomProperty 기반 핸드셰이크와 Quantum의 `SessionRunner.StartAsync` 가 함께 동작.

---

## B1. 단계별 progressive timeout — 한쪽 로딩 느림

**발생 조건**
- 한쪽 기기 사양이 낮거나 디스크 I/O가 느려 씬 로딩이 길어짐.
- 양쪽이 단계별 ready 신호를 교환하며 진행하는 구조.

**증상**
- 빨리 로딩한 쪽은 "상대 기다리는 중" 대기.
- 사용자가 답답해서 강제 종료.

**근본 원인**
- 단일 timeout(예: 60초)만 두면 부분 실패를 구분 못 함.
- 어느 단계에서 멈췄는지 추적 안 되면 진단/복구 어려움.

**대응 — 단계별 progressive timeout**

각 단계마다 별도 timeout과 fallback 액션을 둔다. 권장 단계 구성:

| 단계 신호 | 의미 | timeout (예시) | timeout 시 액션 |
|----------|------|----------------|----------------|
| `sceneReady` | 씬·리소스 로딩 끝 | 7s | 대체 세션 정책 1회 시도 |
| `uiReady` | UI 준비 완료 | 15s | 대체 세션 정책 + 시뮬 단독 시작 |
| `dataReady` | 초기 데이터 적용 완료 | 30s (15s 시점에 1회 재전송) | 네트워크 안내 + 로비 복귀 |
| `gameReady` | Quantum 세션 확인 완료 | 10s + AddPlayer confirm 15s | 등록 실패 → 네트워크 안내 |

> **timeout 값은 게임 정책에 따라 조정**. 위 값은 캐주얼 모바일 멀티 기준. PC 게임이나 고품질 매칭 게임은 더 길게 잡을 수 있음.

**진행률 broadcast**: 각 단계 셋팅 시점에 CustomPlayerProperty `progress` (정수 또는 enum)로 송신 → 상대 화면에 진행 단계 표시 가능.

**문구 단계화**: 본인 쪽 대기 시간이 길어지면 UI 메시지도 단계화. 사용자에게 "버그 아님" 신호.

**상대가 이미 대체 세션으로 전환된 경우**: 대체 정책(봇/AI/무승부 등) 적용 이후엔 상대 쪽 ready 신호가 늦게 도착해도 무시 — "둘 다 준비됨"으로 착각해서 잘못된 시점에 진행하면 안 됨.

**우선순위**: 中

---

## B2. 로딩 중 상대 이탈

**발생 조건**
- 양쪽 모두 씬 로딩 중인데 한쪽이 앱 종료 / 끊김.

**증상**
- `OnPlayerLeftRoom(otherPlayer)` 발화.
- 그러나 Quantum 세션은 아직 시작되지 않았거나 막 시작된 상태.

**근본 원인**
- 로딩 페이즈에서 상대 이탈을 감지하는 책임이 명확하지 않으면 양쪽 모두 이상한 상태가 됨.
- Quantum `DeterministicGameMode.Multiplayer`로 시작된 후엔 PlayerCount 부족으로 세션이 진행되지 않을 수 있음.

**대응**
- `OnPlayerLeftRoom` 콜백을 페이즈별로 분기:
  - Lobby / Result → 무시
  - SceneLoading / Gameplay → 봇 활성화 트리거
- 봇 활성화 시 방 `IsOpen = false`, `IsVisible = false` 로 변경. 재진입 차단.
- Quantum 세션이 시작되기 전이면 `SessionRunner.Arguments`의 `GameMode`를 `Local`로 바꾸고 봇 RuntimePlayer로 재시작. 이미 Multiplayer로 시작했다면 `BotActivateCommand` 같은 deterministic command로 봇 행동을 시뮬레이션에 주입.

**우선순위**: 高

---

## B3. 다단계 ready 동기화 실패

**발생 조건**
- `ready` → `uiReady` → `dataReady` → `gameReady` 같은 단계별 동기화에서 한 단계가 멈춤.
- 예: 한쪽은 UI 준비가 완료됐는데 다른 쪽 디바이스에선 frame drop으로 늦어짐.

**증상**
- 진행 화면이 특정 단계에서 멈춰 있음.
- 양쪽 모두 "상대 기다리는 중".

**근본 원인**
- 단계별 ready 키를 송신하는 코드가 누락됐거나 비동기 흐름이 깨짐.
- CustomProperty 업데이트가 일시적으로 못 전달됨 (네트워크 jitter).

**대응**
- 각 단계 셋팅 시점에 로그 + 단계 timeout (단계당 10~15초).
- timeout 초과 시 상대 단계를 강제로 통과 처리하거나(위험), 매칭을 끊고 결과 화면으로.
- 디버그 로그에 항상 양쪽의 마지막 단계 키 + 본인 단계 키를 함께 기록 → 어느 단계에서 깨졌는지 추적.

**우선순위**: 中

---

## B4. Quantum 세션 시작 실패

**발생 조건**
- `SessionRunner.StartAsync(args)` 호출 시 예외.
- `Communicator` (QuantumNetworkCommunicator) 가 Photon `RealtimeClient`를 잃었거나 disconnect 상태.

**증상**
- 매칭 완료 후 씬 진입했는데 게임이 시작 안 됨.
- `RuntimeException`, `NullReferenceException`, `OperationCanceledException` 등.

**근본 원인**
- `RealtimeClient`가 매칭 완료와 Quantum 시작 사이에 disconnect됨.
- `SessionRunner.Arguments` 필드 누락 (RuntimeConfig, SessionConfig, ClientId 등).
- Quantum `Communicator` 사용 시점에 Photon 채널이 이미 닫혀 있음.

**대응**
- `SessionRunner.StartAsync` 호출 직전에 `_client.IsConnectedAndReady`와 `_client.CurrentRoom != null` 검증.
- 예외를 catch해서 사용자에게 명시 메시지 + 로비 복귀.
- `using (new ConnectionServiceScope(_client))` 같은 스코프 패턴으로 Quantum이 Communicator를 점유하는 동안 다른 코드가 client를 disconnect 못 하게 보호.

**우선순위**: 高

---

## B5. RuntimeConfig 빌드 실패

**발생 조건**
- 게임 시작에 필요한 데이터 (맵, 스폰 위치, 시드 등)를 모아 `RuntimeConfig`를 만드는 단계에서 누락 / 검증 실패.

**증상**
- 한쪽은 정상 빌드, 다른 쪽은 null 반환 → 매칭 자체가 중단되거나 한쪽만 시작.
- 또는 양쪽 다 RuntimeConfig가 달라서 Quantum이 즉시 desync.

**근본 원인**
- 멀티 전용 데이터(시작 자원, 스폰 위치, 보상 등 게임 시작에 필요한 모든 데이터)가 한쪽 디바이스에만 빌드돼 있고 다른 쪽엔 없음.
- AssetGuid가 디바이스마다 다름 (assetdb 동기화 안 됨).

**대응**
- RuntimeConfig 빌드는 **방장이 한 번** 만들고 직렬화해서 Photon RoomProperty로 broadcast → 다른 클라이언트가 deserialize해서 사용. 둘 다 따로 빌드하지 않음.
- 빌드 실패 시 null 반환 → 매칭 중단 + 사용자에게 명시.
- AssetGuid는 buildtime에 결정되도록 strict, ScriptableObject hashing 등으로 검증.

**우선순위**: 高

---

## B6. AddPlayer confirm timeout

**발생 조건**
- `QuantumRunner.Game.AddPlayer(playerSlot, runtimePlayer)` 호출 후 `CallbackLocalPlayerAddConfirmed` 가 일정 시간 내 발화되지 않음.

**증상**
- 본인의 `PlayerRef`를 못 받음.
- 후속 ACK 비교 / Command 송신이 모두 미스매치.

**근본 원인**
- Quantum 시뮬레이션이 시작은 됐지만 본인의 AddPlayer 요청이 deterministic input으로 들어가지 못함.
- 네트워크 지연으로 input snapshot이 누락.

**대응**
- Confirm timeout (권장 10~15초) 설정.
- Fallback: `runner.Game.GetLocalPlayers()`를 호출해 가장 첫 PlayerRef를 사용. 단, Multiplayer 모드에서 이게 빈 리스트면 정말 실패한 것.
- Fallback도 실패 시 매칭 중단 + 사용자에게 "네트워크 환경을 확인해주세요" 안내.

**우선순위**: 高

---

## B7. 초기 데이터 ACK timeout

**발생 조건**
- Quantum 세션 시작 후 플레이어 캐릭터/장비/컴포넌트 등 초기 데이터를 Command로 송신.
- 상대가 적용 완료 신호(`EventOnXxxAppliedEvent` 등)를 일정 시간 내 안 보냄.

**증상**
- 게임은 시작됐는데 한쪽 화면에 본인 데이터(캐릭터/장비 등)가 안 보임.
- 또는 양쪽의 데이터가 어긋남.

**근본 원인**
- Command가 deterministic input slot을 가득 채우거나 frame 격차로 적용이 늦어짐.
- 적용 후 이벤트 발화를 안 하거나, 이벤트 구독 시점이 발화 이후라 놓침.

**대응**
- ACK timeout 30초 (봇 활성화 중이면 +10초 grace).
- 15초 시점에 미도착 항목 1회 재전송.
- 이벤트 구독은 `SessionRunner.StartAsync` 이전에 `QuantumCallback.Subscribe`로 미리 등록 → 첫 발화도 놓치지 않음.
- Command는 양쪽 모두 자신의 데이터를 자신이 송신 (반대편이 송신하면 게임 시작 전 데이터가 안 보임).

**우선순위**: 高

---

## B8. 봇 데이터 fetch 실패 / 부분 실패

**발생 조건**
- 대체 세션(봇/AI 등)으로 fallback될 때 봇/AI의 프로필 데이터(닉네임, 캐릭터, 장비 등)를 서버 API로 가져옴.
- API timeout / 5xx 응답.

> **이 시나리오는 봇/AI 대체 정책을 채택한 게임에만 해당**. 매칭 실패 시 무승부/취소 처리하는 게임은 skip.

**증상**
- 봇/AI 닉네임이 기본값으로 표시.
- 또는 봇/AI 캐릭터가 안 등장 / 기본 캐릭터만 등장.

**근본 원인**
- 외부 API 의존. 매칭 시간과 fetch 시간이 겹치면 사용자 대기가 길어짐.
- 재시도 로직 부족.

**대응**
- 최대 3회 재시도 + 지수 백오프 (1s → 2s → 4s).
- 시도당 timeout 짧게 (3~5초).
- 실패 시 로컬 기본 데이터로 fallback → 게임은 진행되도록.
- API fetch는 매칭 시작 시점에 미리 trigger → 대체 세션 fallback이 결정되기 전에 캐시 완료될 가능성 높임.

**우선순위**: 中

---

## B9. 로딩 중 백그라운드 진입

**발생 조건**
- 양쪽이 씬 로딩 중인데 한쪽이 홈 버튼 / 다른 앱 전환.

**증상**
- 백그라운드 진입 시점에 Photon은 30초 grace로 유지.
- 그 사이에 Quantum 세션이 시작되면 본인 input은 안 들어가고 상대만 게임 진행.
- 30초 이내 복귀하면 Quantum이 catch-up으로 따라잡음. 30초 초과면 disconnect → 봇 활성화.

**근본 원인**
- 로딩 페이즈에선 본인이 input 송신을 해야 시뮬레이션이 진행됨 (PlayerCount=2).
- 백그라운드에서 Unity Update가 멈추거나 throttle됨.

**대응**
- 페이즈가 SceneLoading일 때 `OnApplicationPause(true)` → 백그라운드 진입 시각 기록.
- 복귀 시 elapsed 계산:
  - `< 30s` → "Quantum이 catch-up할 것" 로그, 별도 처리 없음.
  - `>= 30s` → 로컬 재접속 실패 이벤트 발화 → 결과 화면으로.
- Photon `KeepAliveInBackground = 30000` 와 로컬 타이머를 동일한 값으로 맞춤.

**우선순위**: 高

---

## B10. 상대 프로필만 영원히 안 옴 (무한 대기)

**발생 조건**
- 매칭은 성립했고 양쪽 모두 방에 있음 (Photon 연결 정상).
- 한쪽이 본인의 프로필 데이터(닉네임/아이콘/프레임/접두사 등)를 CustomPlayerProperty로 송신해야 하는데 일부 키가 도착하지 않음.

**증상**
- 본인 시점에선 상대 슬롯이 "이름 없음" / 아이콘 없음.
- 프로필 표시 화면이 멈춰 있음 (별도 timeout이 없으면 무한 대기).
- 사용자가 강제 종료할 때까지 화면이 그대로.

**근본 원인**
- 프로필 송수신 코드가 한쪽 키만 보내고 다른 키 송신이 누락됨.
- 또는 송신 시점이 상대 입장 전이라 첫 발화를 놓침.
- `OnPlayerPropertiesUpdate` 핸들러가 일부 키만 처리하고 미도착 키 감지 로직이 없음.
- **별도 timeout이 없으면 사용자에게는 "응답 없는 게임"으로 보임**.

**대응**
- 프로필 송신은 양쪽이 자신 데이터를 자신이 송신. 입장 시점에 즉시 송신.
- 본인이 들어왔을 때 상대가 이미 송신한 properties는 `Player.CustomProperties`로 조회 가능 → 입장 직후 한 번 fetch.
- **프로필 도착 timeout** 설정 (권장 15~20초). 초과 시:
  - 부분 데이터로 진행 (받은 키만 표시, 나머지는 기본값).
  - 또는 봇 활성화 (사용자에게 명시).
- 누락 키 1회 재요청 신호 (`requestProfile = true` CustomPlayerProperty) → 상대가 다시 송신.
- 프로필 누락이 일정 비율 이상 발생하면 텔레메트리.

**우선순위**: 高

---

## B11. 의도적 disconnect 진행 중 늦은 콜백 (race)

**발생 조건**
- 사용자가 매칭 취소 / 게임 종료 / 새 매칭 시작 → `DisconnectAsync()` 호출 중.
- 그 사이에 이전 세션의 콜백(예: `OnPlayerEnteredRoom`, `EventOnSomethingApplied`, AddPlayer confirm)이 늦게 도착.

**증상**
- 새 매칭 / 다음 화면의 상태가 이전 판 데이터로 오염.
- 봇 활성화가 의도치 않게 트리거되거나, 이전 판의 ready 플래그가 새 판에서 true로 인식.

**근본 원인**
- async 흐름에서 disconnect 요청과 콜백 수신이 시간적으로 겹침.
- Quantum/Photon 콜백은 register만 하면 자동 발화 — 별도 가드 없으면 모두 처리됨.

**대응**
- "의도적 disconnect 진행 중" 플래그 (`_isShuttingDown`) 설정 → disconnect 요청 시 true, disconnect 완료 + 새 세션 시작 시 false.
- 모든 콜백 핸들러 첫 줄에 가드:
  ```csharp
  if (_isShuttingDown) return;
  ```
- Quantum 콜백은 `QuantumCallback.UnsubscribeListener(this)` 로 명시적 해제 → 그 후 disconnect.
- 새 세션 시작 시 모든 콜백 재구독.
- 메모리 리셋(A12)과 짝이 됨 — 가드 + 리셋을 함께 적용해야 stale 상태 완전 제거.

**우선순위**: 高

# C. 인게임 (Gameplay phase)

Quantum 세션이 시작되어 양쪽이 실제 플레이 중인 페이즈. 이 페이즈에서의 끊김은 게임 진행 자체에 영향을 줌.

## 대체 세션 정책 (Replacement Policy) — 사전 정의 필요

이 카테고리의 여러 시나리오(C1, C8, C10, C11)는 상대 이탈 후 어떻게 처리할지 게임이 사전에 정의한 정책을 따른다. 흔한 옵션:

| 정책 | 동작 | 적용 게임 예 |
|------|------|--------------|
| **봇/AI 대체** | 이탈자 슬롯을 봇이 인계, 게임 계속 | 캐주얼 모바일 PvP, 협동 디펜스 |
| **즉시 무승부/종료** | 양쪽 모두 결과 화면, 매칭 무효 | 경쟁 매칭, 랭크 게임 |
| **남은 인원으로 계속** | 빈 슬롯 그대로 두고 진행 (3v3 → 2v3) | 4인 이상 게임 |
| **항복 옵션** | 남은 쪽이 선택 — 항복 / 계속 | 일부 PvP |

본 문서의 시나리오는 "대체 세션 정책 실행"으로 추상화해 기술한다. 봇/AI 대체 정책을 채택한 게임이 가장 흔하므로 예시 코드는 그 가정을 따른다. 다른 정책을 쓰는 게임은 "ActivateBot()" 같은 호출을 본인 정책의 대응 함수로 매핑해서 적용.

---

## C1. 상대 이탈 → 봇 활성화

**발생 조건**
- 인게임 중 상대가 앱 종료 / 끊김.
- `OnPlayerLeftRoom(otherPlayer)` 발화.

**증상**
- 양쪽 다 진행 중이던 게임이 한 명만 남음.
- Quantum은 PlayerCount=2로 시작했기 때문에 상대 input이 안 와도 시뮬은 진행 (deterministic, input 없으면 빈 input으로 채움).

**근본 원인**
- 상대가 빠진 후엔 양쪽의 진행이 어긋날 수 있음 (남은 한 명이 모든 게임 액션을 혼자 처리).
- 봇으로 대체하지 않으면 사용자는 "AI가 없는 빈 진영"을 상대로 진행하게 됨.

**대응**
- `OnPlayerLeftRoom`에서 페이즈 확인:
  - Lobby/Result → 무시
  - SceneLoading/Gameplay → `ActivateBot()`
- 봇 활성화 시:
  1. 방 `IsOpen = false`, `IsVisible = false` → 재진입 차단.
  2. 봇 PlayerRef 계산 (로컬의 반대 슬롯).
  3. `BotActivateCommand` 같은 deterministic command 송신 → 시뮬레이션에 봇 행동 주입.
  4. UI에 "상대가 이탈해 AI가 대신합니다" 알림.
- 봇 행동 로직은 사전 정의된 패턴 또는 서버에서 받은 프로필 기반.

**우선순위**: 高

---

## C2. 자신 끊김 → 로컬 재접속 타이머

**발생 조건**
- 본인의 네트워크가 일시적으로 끊김 (지하철 진입, 엘리베이터, Wi-Fi 약해짐).
- `OnDisconnected(DisconnectCause.ClientTimeout)` 발화.

**증상**
- 게임 화면이 멈춤 (Quantum input snapshot이 안 와서).
- 또는 자기 input만 로컬에서 보이고 상대 진행이 안 보임.

**근본 원인**
- 끊김 즉시 게임을 종료하면 일시적 끊김에서 회복할 기회가 없음.
- 끊김 후 무한 대기하면 사용자가 답답해서 강제 종료.

**대응**
- 로컬 재접속 타이머 (권장 30초, Photon `KeepAliveInBackground`와 동일).
- 타이머 동안 UI 오버레이 "재연결 시도 중... (XX초 남음)" 표시.
- `Application.internetReachability != NotReachable` 일 때 `ReconnectAndRejoin()` 시도.
- 타이머 만료 시 결과 화면으로 (게임 패배 또는 무승부 처리).
- 끊김 중 상대는 봇 활성화될 수 있음 (상대가 본인을 이탈로 봄). 복귀 시 상대 시점에선 본인이 다시 들어왔다고 인식할 수 있도록 처리 (C8 참조).

**우선순위**: 高

---

## C3. Photon Disconnect — 사유별 분기

**발생 조건**
- `OnDisconnected(DisconnectCause cause)` 발화.

**증상**
- 사유에 따라 사용자 대응 / 자동 처리가 달라야 함.

**근본 원인**
- `DisconnectCause`는 16개+ 값. 같은 "끊김"이라도 의미가 다름.

**대응**
- 사유별 분기 권장:

| DisconnectCause | 의미 | 대응 |
|------------------|------|------|
| `None` | 미초기화 | 무시 |
| `DisconnectByClientLogic` | 정상 종료 | 메시지 표시 안 함 |
| `DisconnectByServerLogic` | 서버 측 종료 (방 closed 등) | "방이 종료되었습니다" |
| `DisconnectByServerReasonUnknown` | 서버에서 끊었지만 사유 모름 | "서버 오류" |
| `ClientTimeout` | 클라가 timeout 감지 (네트워크 약함) | 재접속 시도 |
| `ServerTimeout` | 서버가 클라 응답 없다고 판단 | 재접속 시도 |
| `Exception` | 코드 예외로 끊김 | 로그 + 결과 화면 |
| `InvalidAuthentication` | AppId/토큰 문제 | 앱 재시작 안내 |
| `MaxCcuReached` | Photon plan CCU 초과 | "잠시 후 다시 시도해주세요" |
| `InvalidRegion` | Region 설정 오류 | 자동 region 재선택 |
| `OperationLimitReached` | OpRequest 너무 많음 | 클라 rate-limit 점검 |
| `DisconnectByOperationLimit` | Operation 차단 | 동일 |
| `DisconnectByDisconnectMessage` | 서버 명령 | 메시지 표시 |
| `ApplicationQuit` | 앱 종료 | 무시 |
| `AuthenticationTicketExpired` | 인증 만료 | 재인증 |
| `ServerAddressInvalid` | 서버 주소 잘못됨 | 앱 재시작 안내 |

- `OnDisconnected` 핸들러가 페이즈와 사유를 함께 보고 결정하도록 작성.

**우선순위**: 高

---

## C4. Checksum 에러 (Desync)

**발생 조건**
- Quantum이 frame N의 simulation state hash를 비교했을 때 클라이언트별로 다름.
- `CallbackChecksumErrorFrameDump` 발화.

**증상**
- 인게임 진행이 어긋남 (한쪽엔 적이 죽었는데 다른 쪽엔 살아있음).
- Quantum이 시뮬레이션을 멈추거나 자동 복구를 시도.

**근본 원인**
- Quantum 외부 의존(예: `UnityEngine.Random`, `DateTime.Now`, `Time.deltaTime` 등 deterministic하지 않은 값)을 시뮬레이션에서 사용.
- AssetGuid 불일치.
- 한쪽만 다른 ScriptableObject / 설정 데이터 버전 사용.
- 시뮬레이션에서 `IEnumerable` 순회 시 정렬되지 않은 컬렉션 사용 (Dictionary 등).

**대응**
- 즉각적 처리: `CallbackChecksumErrorFrameDump` 구독 → frame dump 저장 + 사용자에게 "동기화 오류" 알림 후 결과 화면으로.
- 근본 처리(개발 단계):
  1. 시뮬레이션 코드(Quantum systems)에서 비결정성 원천 제거.
  2. `Quantum.Frame.RNG`만 사용.
  3. `Time.deltaTime` → `Frame.DeltaTime`.
  4. Asset 변경 시 양쪽 디바이스 모두 최신 버전 보장 (앱 버전 강제 업데이트).
- 빈도 모니터링: checksum 에러 발생률을 텔레메트리로 수집 → 임계 초과 시 release 차단.

**우선순위**: 高

---

## C5. 백그라운드 30초 초과

**발생 조건**
- 인게임 중 백그라운드 진입 후 30초 이상 경과.
- `OnApplicationPause(false)` 복귀 시점에 elapsed >= 30s.

**증상**
- Photon이 자동으로 disconnect → `OnDisconnected(ClientTimeout)`.
- 상대 입장에선 본인이 이탈한 것으로 보임 → 봇 활성화됐을 가능성.

**근본 원인**
- iOS/Android는 백그라운드 앱의 네트워크 활동을 제한.
- Photon `KeepAliveInBackground = 30000` 이 한계.

**대응**
- `OnApplicationPause(true)` 진입 시각 기록.
- `OnApplicationPause(false)` 복귀 시 elapsed 계산:
  - `>= 30s` → 로컬 재접속 실패 처리 → 결과 화면 (게임 종료).
  - `< 30s` → Quantum catch-up 시도.
- 사용자에게 "백그라운드 시간이 너무 길어 게임에서 이탈되었습니다" 안내.
- 결과 화면에선 본인의 보상이 어떻게 처리되는지 명확히 표시 (패배 처리, 보상 일부 지급 등 게임 정책).

**우선순위**: 高

---

## C6. 백그라운드 30초 이내 복귀

**발생 조건**
- 인게임 중 잠깐 다른 앱 전환 후 곧 복귀 (예: 알림 확인).

**증상**
- 복귀 시 Photon 연결은 유지됨.
- Quantum 시뮬레이션이 frame 격차 따라잡기(catch-up) 시작.
- 사용자 화면은 잠시 빠르게 진행되는 것처럼 보임.

**근본 원인**
- Quantum의 prediction/verification 메커니즘 — 늦은 input은 verified frame이 따라잡힘.
- 백그라운드 동안 Unity Update가 멈춰서 frame이 밀림.

**대응**
- 별도 처리 거의 필요 없음. Quantum이 자체적으로 catch-up.
- 단, 사용자에게 catch-up 동안 UI 오버레이 ("동기화 중...") 표시하면 UX 개선.
- `QuantumRunner.GameFlags`로 catch-up 상태 확인 가능.
- catch-up 중엔 사용자 input 막을 것 (잘못 누른 input이 빠르게 적용되어 사용자가 혼란).

**우선순위**: 中

---

## C7. Wi-Fi ↔ Cellular 전환

**발생 조건**
- 인게임 중 Wi-Fi 신호 사라짐 → 자동으로 LTE/5G로 전환.
- 반대로 Wi-Fi 영역 진입 시 자동 전환.

**증상**
- 짧은 패킷 손실 (1~3초).
- Photon은 보통 ClientTimeout 안 잡고 견딤. 하지만 일부 OS에선 socket이 끊김.

**근본 원인**
- IP 주소가 바뀌면 TCP socket은 끊김 (UDP는 ip 변경에도 유지될 수 있음).
- iOS는 Wi-Fi/Cellular dual connection 시 자동 전환을 잘 처리.
- Android는 디바이스/OS 버전마다 다름.

**대응**
- Photon 프로토콜은 가능하면 UDP 사용 (`AppSettings.Protocol = ConnectionProtocol.Udp`).
- 끊김 발생 시 C2의 재접속 흐름으로 처리.
- 텔레메트리: `NetworkReachability` 변화를 로그 → 끊김 빈도와 원인 추적.

**우선순위**: 中

---

## C8. 봇 활성화 후 원래 상대 복귀

**발생 조건**
- 상대 일시 끊김 → 봇 활성화 → 상대가 30초 이내 복귀.

**증상**
- 상대가 방에 다시 들어옴 (`OnPlayerEnteredRoom` 발화, ActorNumber 동일).
- 본인 시점엔 이미 봇이 행동 중.

**근본 원인**
- 봇과 원래 상대의 행동이 중복되거나 충돌.
- Quantum 시뮬레이션은 deterministic — 봇 활성화 command를 이미 발행한 상태라 상대가 복귀해도 자동 처리되지 않음.

**대응**
- `OnPlayerEnteredRoom` 시 ActorNumber가 직전 이탈자(`_opponentActorNumber`)와 같으면 "복귀"로 인식.
- 복귀 인식 시 `BotDeactivateCommand` 송신 → 봇 행동 중단.
- 단, 게임 정책상 봇 활성화 후엔 복귀 불가하게 막을 수도 있음. 이때 방을 IsOpen=false로 닫아두면 상대는 다시 못 들어옴.
- 정책 결정 필요: "복귀 허용 vs 봇 fix". 양쪽 다 일관되게 적용.

**우선순위**: 中

---

## C9. Command 전송 실패

**발생 조건**
- Quantum `DeterministicCommand`를 `runner.Game.SendCommand(...)`로 보냄.
- runner가 null이거나 disconnect 상태.

**증상**
- Command가 시뮬레이션에 도달하지 않음.
- 게임 상태가 한쪽에서만 변함 또는 양쪽 다 변하지 않음.

**근본 원인**
- Command sender가 게임 종료 후에도 호출됨.
- runner 참조가 stale (이전 세션의 runner를 들고 있음).
- 봇 게임(Local 모드)에서도 command 송신이 필요한데 multiplayer runner 참조만 들고 있음.

**대응**
- Command sender는 항상 `QuantumRunner.Default` 또는 매니저가 보관 중인 최신 runner 참조 사용.
- 송신 직전에 `runner != null && runner.IsRunning` 검증.
- 봇 게임으로 전환 시 새 runner 참조를 매니저에 등록.
- 송신 실패 시 retry 또는 무시 (Command가 결정적이지 않은 input이라면 무시해도 결정론에 영향 없음).

**우선순위**: 中

---

## C10. 동시 disconnect — 양쪽 다 봇 활성화 시도

**발생 조건**
- 양쪽 클라이언트가 거의 동시에 끊김 / 백그라운드.
- 한쪽 복귀, 다른 쪽은 영구 이탈.

**증상**
- 복귀한 쪽이 봇 활성화를 시도하는데 상대가 이미 사라진 상태.
- 또는 양쪽 다 30s 후 복귀해서 봇이 양쪽에서 활성화됨.

**근본 원인**
- 봇 활성화는 로컬 결정인데 deterministic command로 시뮬에 영향을 줌.
- 한쪽이 봇 활성화 command를 보냈는데 상대가 안 받으면 양쪽 시뮬이 어긋남.

**대응**
- 봇 활성화 책임은 항상 **이탈하지 않은 쪽**이 가짐 → `OnPlayerLeftRoom`에서 본인이 활성화 결정.
- Command를 보낸 시점의 frame을 시뮬에 기록 → catch-up 후에도 양쪽이 동일 frame에서 봇 활성화.
- 양쪽이 동시에 봇 활성화 command를 보내면 시뮬레이션이 "두 봇" 케이스로 처리 — 이건 상태가 비합리적이지만 결정적임. 게임 정책으로 "두 봇 = 게임 종료" 처리 가능.

**우선순위**: 中

---

## C11. AI 판 (Local 모드) 중 실시간 방 통신 비활성

**발생 조건**
- 매칭 실패 또는 상대 이탈로 봇 게임(Quantum `Local` 모드)으로 진행 중.
- 그러나 이전에 열려 있던 Photon `RealtimeClient` 객체가 남아 있음.

**증상**
- 봇 게임 중인데 백그라운드에서 Photon ping이 계속 돔.
- Photon이 빈 방을 keep-alive하면서 매칭 풀에 "유령 방"으로 노출되거나, 다른 사용자가 그 방에 들어와 시뮬이 꼬임.
- 봇 게임 종료 후 다음 매칭 시 client 상태가 stale.

**근본 원인**
- 봇 게임은 Quantum `Local` 모드 — 네트워크 입력이 필요 없음.
- 그러나 Photon client 객체가 살아있으면 자체 통신 루프 (ping, 방 properties 동기화 등)는 계속 돔.

**대응**
- 봇 게임 진입 시점에:
  1. 현재 방을 `IsOpen = false`, `IsVisible = false` 로 변경 (재진입 차단).
  2. `await _client.DisconnectAsync()` — Photon 연결 완전 종료.
  3. `_client` 참조도 null로 (next match에서 새로 생성).
- 봇 게임이 진행되는 동안 어떤 Photon 콜백도 발화 안 함 → 봇 활성화 command만 로컬 시뮬에 직접 송신.
- 봇 게임 종료 후 로비 복귀 시 다시 매칭 entry로 → A12 메모리 리셋 적용.

**우선순위**: 中

---

## C12. 재접속 후 슬롯 확정 — 로컬 UI 재동기화

**발생 조건**
- 인게임 중 끊김 → grace timeout 이내 재접속 성공.
- Quantum 시뮬레이션은 catch-up 으로 따라잡음.
- 그러나 로컬 UI (점수/체력/자원/슬롯 표시 등 Quantum 외부 UI 값)는 끊김 직전 값으로 멈춰 있음.

**증상**
- 재접속 후 게임은 정상 진행되지만 UI 숫자가 안 맞음.
- 플레이어 슬롯에 따른 화면 (예: 카메라 위치/방향, 진영 표시) 이 잘못된 슬롯 기준으로 보임.
- 한두 액션 후엔 자동 보정되지만 사용자에게 어색.

**근본 원인**
- UI는 Quantum 외부 — `Frame.Verified` 변화를 이벤트/시그널로 받아 갱신.
- 끊김 동안 이벤트가 누락된 경우 마지막 값 유지.
- 슬롯별 화면 분기 (카메라/UI 좌우/진영 색깔 등) 는 본인 PlayerRef 기준으로 결정되는데, 재접속 시점에 PlayerRef 확인이 늦으면 기본값(첫 슬롯) 으로 표시.

**대응**
- 재접속 성공 (시뮬에 플레이어가 다시 붙음) 시점에 **로컬 UI 강제 재동기화** 호출:
  - `Frame.Get<TComponent>(playerEntity)` 로 현재 상태 조회 → UI 갱신
  - 본인 PlayerRef 재확인 → 슬롯별 화면 분기 (카메라/진영/UI 좌우 등) 다시 적용
  - 점수, 체력, 자원, 게임 액션 표시 등 모든 로컬 표시 값
- 재동기화 트리거는 `CallbackLocalPlayerAddConfirmed` 또는 별도 "재접속 완료" 이벤트.
- 재동기화 도중 사용자 input은 잠깐 잠금 (1~2 frame).

**우선순위**: 中

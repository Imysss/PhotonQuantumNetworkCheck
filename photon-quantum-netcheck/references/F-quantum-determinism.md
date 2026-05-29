# F. Quantum 결정론

Photon Quantum의 핵심은 결정론(determinism). 같은 input + 같은 RuntimeConfig + 같은 시뮬레이션 코드 = 모든 클라이언트에서 같은 결과. 이 보장이 깨지면 desync.

---

## F1. RuntimeConfig 불일치

**발생 조건**
- 양쪽 클라이언트가 서로 다른 `RuntimeConfig`로 `SessionRunner.StartAsync` 호출.

**증상**
- 시작 즉시 desync 또는 첫 frame부터 checksum 에러.
- 한쪽엔 스폰된 엔티티가 다른 쪽엔 없음.

**근본 원인**
- 양쪽이 독립적으로 RuntimeConfig를 빌드 → 시드, 엔티티 풀, 맵 변형 등이 다름.
- AssetGuid가 디바이스마다 다름 (Asset DB 동기화 안 됨).

**대응**
- RuntimeConfig는 **한쪽이 빌드 → 다른 쪽에 broadcast**:
  - 옵션 1: 방장(MasterClient)이 빌드 → Photon RoomProperty로 직렬화 전송.
  - 옵션 2: `SessionRunner.Arguments`의 `RuntimeConfig`는 Quantum이 자동으로 동기화하므로 동일한 인스턴스를 사용하면 됨. 하지만 두 클라이언트가 다른 `RuntimeConfig` 객체를 args에 넣으면 동기화되지 않음. Quantum은 시작 시 한쪽의 config를 채택.
- 빌드 시 사용하는 모든 데이터가 deterministic인지 검증:
  - Random seed → `RuntimeConfig.Seed`에 명시.
  - AssetRef는 AssetGuid 기준 → 양쪽이 동일한 asset bundle 가지고 있어야 함.
- 디버그 모드에선 양쪽이 빌드한 RuntimeConfig hash를 비교 → 불일치 즉시 로그.

**우선순위**: 高

---

## F2. Asset / 데이터 버전 차이

**발생 조건**
- 한쪽은 v1.9.7 앱, 다른 쪽은 v1.9.8 앱.
- 두 버전에서 같은 엔티티(캐릭터/유닛/아이템 등)의 스탯이 다름.

**증상**
- 시뮬레이션 진행 중 어느 시점에 desync.
- "어제까지 잘 되던 게임이 오늘 갑자기 끊김" 신고.

**근본 원인**
- Quantum 시뮬레이션 코드가 ScriptableObject / 설정 asset의 값을 참조하는데 양쪽의 데이터가 다름.
- AssetGuid는 같지만 내부 값이 다른 케이스 (StreamingAssets 또는 Addressable로 업데이트했을 때).

**대응**
- 매칭 시 양쪽의 앱 버전을 Photon CustomPlayerProperty로 교환 → 다르면 매칭 자체를 거부.
- 강제 업데이트 시스템: 서버에서 최소 지원 버전을 받아 그 미만이면 앱 시작 시 업데이트 강제.
- 멀티 모드에선 빌드된 binary asset만 사용 (StreamingAssets / Addressable 동적 다운로드 금지).
- AssetGuid + content hash를 시작 시 비교 → 불일치 시 매칭 차단.

**우선순위**: 高

---

## F3. Late Join / Snapshot

**발생 조건**
- 게임이 이미 진행 중인 방에 새 플레이어가 들어옴 (Late Join).
- Quantum이 snapshot을 송신해서 새 플레이어가 현재 frame으로 점프해야 함.

**증상**
- Late Joiner가 snapshot을 못 받거나 받았는데 적용 실패.
- 새 플레이어 화면이 처음부터 시작하거나 검은 화면.

**근본 원인**
- Quantum `SessionRunner.Arguments.GameMode = Multiplayer` 에서 late join 시 `Communicator`가 snapshot 요청.
- 큰 snapshot은 송신 시간이 길고 packet loss 시 누락 가능.

**대응**
- 시뮬레이션 상태를 작게 유지 — 큰 컴포넌트 (예: 거대한 list)는 deterministic하게 재구성 가능하도록 설계.
- `ICallbackSimulateFinished` 또는 `ICallbackSnapshot*` 콜백으로 snapshot 송수신을 직접 제어 가능.
- Late join 차단 정책도 선택지: 고정 인원 매칭이라면 시작 후 추가 입장 차단 (`Room.IsOpen = false`).
- 대체 세션(C1) 활성화 후 원래 상대 복귀 시(C8) snapshot이 필요할 수 있음.

**우선순위**: 中

---

## F4. Frame catch-up 격차

**발생 조건**
- 한쪽이 일시적으로 느려져서 verified frame이 뒤처짐.
- 다른 쪽은 prediction frame을 계속 진행.

**증상**
- 느린 쪽 화면이 갑자기 빠르게 진행되는 것처럼 보임 (catch-up).
- 또는 빠른 쪽의 prediction이 틀려서 rollback → 화면이 잠시 깜빡임.

**근본 원인**
- Quantum의 prediction/verification 모델 — `Frame.Predicted`와 `Frame.Verified` 사이에 격차 발생.
- 격차가 너무 크면(default 60 frames) 시뮬레이션 멈추거나 강제 disconnect.

**대응**
- `SessionConfig.RollbackWindow` 조정 (default 60). 모바일 환경에선 더 크게 (90~120) 잡으면 disconnect 줄어듦.
- catch-up 동안 input lock — 사용자가 잘못 누른 input이 빠르게 적용되는 것 방지.
- `QuantumRunner.GameFlags` 또는 `Game.Session.PredictedFrames` / `VerifiedFrames` 차이 모니터링.
- 격차가 임계 초과 시 UI 오버레이 "동기화 중" 표시.

**우선순위**: 中

---

## F5. PauseSimulation / ResumeSimulation

**발생 조건**
- 클라가 명시적으로 `runner.Session.Pause()` 호출.
- 또는 게임 정책상 일시정지 (예: 인터럽트 처리).

**증상**
- Pause한 쪽만 멈춤. 상대는 계속 진행 → 격차 누적.
- Resume 시 한 번에 catch-up 시도 → 큰 점프.

**근본 원인**
- Quantum의 `Pause()`는 로컬 시뮬레이션만 멈춤. 상대는 계속 input을 보내고 시뮬을 진행.
- 멀티 모드에선 사용자가 pause해도 게임이 멈추지 않는 게 정상.

**대응**
- 멀티 모드에선 **로컬 pause를 사용하지 말 것**. 인터럽트 (전화, 알림 등)는 백그라운드 처리로 (E1, E3, E6).
- 양쪽 동시 pause가 필요하면 deterministic command로 시뮬레이션 내부에서 pause 상태 표현 (예: `GamePausedFlag`).
- 싱글플레이 / 로컬 모드에선 `runner.Session.Pause()` 사용 가능.

**우선순위**: 中

# E. 모바일 라이프사이클

OS가 앱의 생명주기에 개입하는 모든 상황. Unity의 `OnApplicationPause` / `OnApplicationFocus` / `OnApplicationQuit` 으로 받지만 OS마다 호출 시점과 보장이 다름.

---

## E1. iOS 전화 수신 / 응답

**발생 조건**
- iOS에서 인게임 중 전화 수신 → 사용자가 응답.
- 시스템 UI가 위로 올라옴.

**증상**
- 응답하지 않으면: `OnApplicationFocus(false)` 발화 (iOS 12+에서 `OnApplicationPause` 대신 focus만 옴).
- 응답하면: `OnApplicationPause(true)` 발화 (앱이 백그라운드로).

**근본 원인**
- iOS의 transient interruption은 focus만 잃고 pause되지 않음.
- 사용자가 통화 거절하면 즉시 focus 복귀.

**대응**
- `OnApplicationFocus(false)` 만 발화된 경우엔 게임을 일시정지하지 않는다 (잠깐의 인터럽트로 게임이 끊기면 안 됨).
- `OnApplicationPause(true)`가 발화되면 인게임 페이즈에 따라 처리 (C5/C6 참조).
- iOS / Android 동작 차이를 핸들러에서 명시:
  ```csharp
  void OnApplicationFocus(bool focus) {
    // iOS transient interrupt — 무시
  }
  void OnApplicationPause(bool pause) {
    // 실제 백그라운드 처리
  }
  ```

**우선순위**: 中

---

## E2. Android 메모리 부족 OOM kill

**발생 조건**
- Android에서 백그라운드 진입 후 다른 무거운 앱들이 메모리 점유.
- OS가 본인 앱을 강제 종료.

**증상**
- 백그라운드 진입 후 일정 시간 뒤 앱 재실행하면 메인 화면부터 시작 (인게임 상태 복원 불가).
- Photon은 클라이언트가 사라진 줄 모르고 PlayerTtl이 다할 때까지 방에 본인을 유지.

**근본 원인**
- OOM kill은 클라이언트에게 사전 알림 없음.
- 앱 재실행 시 이전 세션 정보가 없음.

**대응**
- 백그라운드 진입 시 현재 상태(매칭 중/인게임/결과 등)를 PlayerPrefs 또는 캐시에 저장 → 재실행 시 복원.
- 복원 시 Photon 재접속이 가능한지 확인 — 일반적으로 30초 grace 초과면 재접속 불가, 결과 화면으로 직행.
- 게임 시작 시 메모리 사용량을 최소화 (대용량 asset은 인게임 진입 후 로드, 백그라운드 진입 직전 해제).

**우선순위**: 中

---

## E3. 화면 잠금 / 꺼짐

**발생 조건**
- 사용자가 전원 버튼 눌러 화면 잠금.
- 또는 자동 잠금.

**증상**
- iOS: `OnApplicationPause(true)` 발화.
- Android: 디바이스/OS 버전에 따라 `OnApplicationFocus(false)` 만 발화하기도 함.
- 잠금 상태에서도 일정 시간 앱 프로세스는 살아있음 (백그라운드 처리와 동일).

**근본 원인**
- 화면 잠금은 명시적 "백그라운드 진입"이 아니지만 게임 진행은 사실상 불가능.
- 사용자가 잠금 해제 후 돌아오는 시간 예측 불가.

**대응**
- C5/C6와 동일한 로직으로 처리 (백그라운드 진입 시각 기록 → 복귀 시 elapsed 확인).
- 잠금 직전에 일시정지 UI를 잠깐이라도 표시할 수 있도록 `OnApplicationPause(true)` 처리에 우선순위.
- 잠금 중에는 Application.runInBackground = true로 설정해도 OS가 강제 throttle.

**우선순위**: 中

---

## E4. 비행기 모드 토글

**발생 조건**
- 인게임 중 사용자가 비행기 모드 ON (실수 또는 의도적).

**증상**
- 즉시 모든 네트워크 차단.
- Photon이 ClientTimeout 감지에 ~10초 소요.
- 그 사이엔 사용자 input은 로컬에만 보이고 상대 진행은 멈춤.

**근본 원인**
- 비행기 모드는 OS 레벨에서 socket을 차단.
- `NetworkReachability`는 거의 즉시 `NotReachable`로 바뀜.

**대응**
- 매 N초(2초 권장) `Application.internetReachability` 폴링.
- `NotReachable` 감지 즉시 사용자에게 알림 + 재접속 타이머 시작 (C2와 동일).
- 사용자가 비행기 모드를 끄면 `internetReachability`가 `ReachableViaCarrierDataNetwork` 또는 `ReachableViaLocalAreaNetwork`로 변함 → 자동 재접속 시도.

**우선순위**: 中

---

## E5. 앱 강제 종료 후 재실행

**발생 조건**
- 사용자가 앱을 스와이프해서 종료 → 즉시 다시 실행.

**증상**
- 새 프로세스 시작. Photon client / Quantum runner 모두 새 인스턴스.
- 이전 게임이 있던 방의 정보는 사라짐.

**근본 원인**
- 강제 종료는 `OnApplicationQuit`이 호출되긴 하지만 끝까지 실행될 보장이 없음 (특히 iOS).
- 새 프로세스에선 이전 상태를 알 수 없음.

**대응**
- 강제 종료 후 재실행은 "신규 진입"으로 처리. 이전 게임 결과는 서버에서 조회 (게임 세션 ID로).
- 강제 종료 직전 보상 ACK가 안 된 경우 서버에서 "패배 처리 + 부분 보상" 같은 정책 결정.
- `OnApplicationQuit`에 의존하지 말 것 (실행 보장 없음). 중요한 상태는 진입 시점에 미리 저장.

**우선순위**: 中

---

## E6. 시스템 알림 / 푸시로 포커스 빼앗김

**발생 조건**
- 인게임 중 다른 앱의 푸시 알림이 표시되어 사용자가 그 알림을 탭함.
- 다른 앱으로 전환됨.

**증상**
- `OnApplicationPause(true)` → 백그라운드 진입과 동일.
- 사용자는 잠시 후 본인 앱으로 돌아올 가능성 높음.

**근본 원인**
- 알림 탭은 의도적이지 않은 인터럽트일 수 있음.
- 사용자 입장에선 본인 의지로 끊은 게 아니라 답답할 수 있음.

**대응**
- 일반 백그라운드와 동일 처리 (C5/C6).
- 단, 게임 중요 시점(보스전, 결승 등)에는 알림 차단을 권장하는 UI 안내 가능 (선택사항).
- iOS 14+ / Android 13+ 의 알림 권한 설정으로 사용자가 직접 알림을 제어하도록 유도.

**우선순위**: 低

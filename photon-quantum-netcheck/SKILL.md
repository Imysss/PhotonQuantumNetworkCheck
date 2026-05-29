---
name: photon-quantum-netcheck
description: Photon Quantum 3 멀티플레이 네트워크 문제 진단 및 시나리오 체크리스트. 매칭(랜덤/방코드), 로딩 동기화, 인게임 끊김/재접속, 모바일 라이프사이클(백그라운드/포커스/네트워크 전환), 결과 화면, Quantum 결정론(Desync/Snapshot/Late Join), 매칭 전 사전 검사(에너지/버전), UX gap(조용히 실패하는 케이스) 까지 다룰 때 호출. 명시 호출 전용 — "매칭 진단", "네트워크 진단", "/photon-quantum-netcheck", "끊김 처리 체크리스트" 등 사용자 직접 요청 시에만 활성화.
---

# Photon Quantum Netcheck

Photon Realtime + Quantum 3 기반 멀티플레이 클라이언트의 네트워크 이슈 진단과 대응 시나리오 카탈로그.

## 카테고리별 호출 (args 분기)

이 스킬은 args를 받아 카테고리/시나리오 단위로 응답한다. 사용자가 인자를 전달하면 **해당 영역만** 로드해 응답.

| args 예시 | 동작 |
|----------|------|
| (없음) | 전체 카탈로그 색인 + 두 모드 안내 |
| `A` / `matchmaking` / `매칭` | A 카테고리(매칭) reference 파일만 읽고 응답 |
| `B` / `loading` / `로딩` | B 카테고리(로딩 동기화) |
| `C` / `in-game` / `인게임` | C 카테고리(인게임) |
| `D` / `result` / `결과` | D 카테고리(결과) |
| `E` / `lifecycle` / `라이프사이클` | E 카테고리(모바일 라이프사이클) |
| `F` / `determinism` / `결정론` | F 카테고리(Quantum 결정론) |
| `G` / `pre-match` / `사전검사` | G 카테고리(매칭 진입 전) |
| `H` / `ux-gap` / `조용히실패` | H 카테고리(UX gap — 로그만 남는 케이스) |
| `A6` / `B11` / 임의 시나리오 ID | 해당 시나리오 하나만 발생 조건/증상/근본 원인/대응/우선순위로 응답 |
| `diag` / `진단` | 진단 모드 진입 (코드 스캔 후 우선순위 표 출력) |
| `checklist` / `체크리스트` | 체크리스트 모드 (`checklists/` 파일 안내) |
| `matrix` / `매트릭스` | 본 파일 하단의 시점×이탈 주체 매트릭스 응답 |

**규칙:**
- 카테고리/ID args가 주어지면 **그 파일만** 읽고 응답한다. 다른 카테고리 파일은 읽지 않는다 (context 절약).
- args 단어가 한글/영어 둘 다 인식. 부분 일치 허용 (`매칭중` → A).
- args가 모호하면 우선 색인을 보여주고 사용자에게 다시 묻는다.

## 게임 정책 가정 (기본값과 다르면 해석해서 적용)

이 스킬은 다음을 기본 가정으로 시나리오를 기술한다. 본인 프로젝트가 다르면 그에 맞춰 해석:

| 가정 | 기본값 | 다른 게임에서는 |
|------|--------|----------------|
| 대체 세션 정책 | 봇/AI로 슬롯 인계 → 게임 계속 | 무승부/즉시 종료, 빈 슬롯 진행, 항복 옵션 등 — C 카테고리 도입부 참조 |
| 재접속 grace timeout | 30초 (Photon `KeepAliveInBackground` 와 동일) | 5초~5분 사이로 게임 정책에 따라 조정 |
| 매칭 타임아웃 | 9~13초 (캐주얼 모바일 매칭 풀 기준) | 경쟁/랭크 게임은 분 단위 |
| 매칭 인원 | 2인 1:1 또는 2인 협동 | 4인/6인/팀전은 인원 수 / 슬롯 분기 / Late Join 정책 재해석 |
| 사전 검사 (G) | 자원(에너지/스태미너) + 버전 + 시즌 데이터 | 자원 없는 게임은 G1 skip, PC 게임은 일부 skip |
| 모바일 라이프사이클 (E) | Android/iOS 백그라운드/포커스/OOM 케이스 | PC/콘솔은 E 카테고리 거의 무용 |

특정 시나리오에서 정책이 다른 가정으로 흘러야 하면 그 시나리오의 "대응" 섹션 첫 줄에 단서 추가 (예: "봇/AI 대체 정책을 채택한 게임에만 해당").

## 두 가지 모드

### 1. 진단 모드 (Diagnose)
- 트리거: "이 매칭 코드 진단해줘", "네트워크 처리 점검해줘", "끊김 핸들링 빠진 거 찾아줘", args `diag`
- 절차: `diagnostics/procedure.md` 참고
- 출력: **우선순위 표 + 권장 조치** (아래 형식)

### 2. 체크리스트 / 레퍼런스 모드 (Reference)
- 트리거: "매칭 중 백그라운드 가면 어떻게 처리해야 해?", "방 코드 매칭 시나리오 정리해줘", args `A` / `B11` 등
- 절차: 해당 시나리오를 `references/`에서 찾아 응답

## 진단 모드 출력 형식 (필수)

진단 모드에서는 반드시 아래 표 형식으로 출력하고, 표 아래에 "권장 조치" 섹션을 둔다.

| ID | 시나리오 | 상태 | 심각도 | 근거 (파일:라인) | 권장 조치 |
|----|----------|------|--------|-------------------|-----------|
| A6 | 매칭 중 백그라운드 진입 | 미구현 | 高 | (없음) | OnApplicationPause(true)에서 Disconnect 호출 |
| C2 | 자신 끊김 → 로컬 재접속 타이머 | 부분 | 中 | Foo.cs:123 | ReconnectAndRejoin 호출 + UI 오버레이 |

- **상태**: `OK` / `부분` / `미구현` / `불확실`
- **심각도**: `高` (게임 진행 불가) / `中` (UX 손상) / `低` (드물거나 회복 가능)
- **근거**: 진단 모드에서 코드를 읽었을 때 파일 경로 + 라인. 못 찾았으면 `(없음)`.

표는 심각도 → 상태(미구현 우선) → ID 순으로 정렬.

## 시나리오 카탈로그 (8 카테고리)

각 시나리오는 5필드 구조: **발생 조건 / 증상 / 근본 원인 / 대응 / 우선순위**.

| 카테고리 | 파일 | 시나리오 수 |
|----------|------|-------------|
| A. 매칭 (Lobby phase) | [references/A-matchmaking.md](references/A-matchmaking.md) | 13 |
| B. 로딩 동기화 (SceneLoading phase) | [references/B-loading-sync.md](references/B-loading-sync.md) | 11 |
| C. 인게임 (Gameplay phase) | [references/C-in-game-disconnect.md](references/C-in-game-disconnect.md) | 12 |
| D. 결과 (Result phase) | [references/D-result-phase.md](references/D-result-phase.md) | 4 |
| E. 모바일 라이프사이클 | [references/E-mobile-lifecycle.md](references/E-mobile-lifecycle.md) | 6 |
| F. Quantum 결정론 | [references/F-quantum-determinism.md](references/F-quantum-determinism.md) | 5 |
| G. 매칭 진입 전 (Pre-match) | [references/G-pre-match.md](references/G-pre-match.md) | 4 |
| H. UX gap (조용히 실패) | [references/H-ux-gap.md](references/H-ux-gap.md) | 6 |

## 체크리스트 (구현·PR 점검용)

| 용도 | 파일 |
|------|------|
| 매칭 기능 PR | [checklists/matchmaking.md](checklists/matchmaking.md) |
| 인게임 네트워크 처리 | [checklists/in-game.md](checklists/in-game.md) |
| 앱 라이프사이클 처리 | [checklists/lifecycle.md](checklists/lifecycle.md) |
| 매칭 진입 전 검사 | [checklists/pre-match.md](checklists/pre-match.md) |
| UX gap 점검 | [checklists/ux-gap.md](checklists/ux-gap.md) |

## 진단 절차

상세 절차는 [diagnostics/procedure.md](diagnostics/procedure.md).

요약:
1. 사용자에게 진단 범위 확인 (특정 파일 / 매칭 코드 / 전체).
2. 해당 범위의 코드를 읽고 시나리오 ID별로 매핑.
3. 표 작성 → 미구현·부분 항목 위주로 권장 조치 제시.
4. **반드시** 표 아래에 "다음 단계" 한 줄 — 사용자가 어디부터 손대면 좋을지.

## 시점 × 이탈 주체 매트릭스

상대가 이탈했을 때 / 본인이 끊겼을 때 페이즈별 처리 요약. 한눈에 보는 결정 트리.

| 페이즈 (플레이어 관점) | 상대가 나감 | 본인이 끊김 |
|------------------------|-------------|--------------|
| 매칭 대기 (1명) | 봇 활성화 X — 매칭 타이머 계속 | 매칭 실패 → 팝업 닫기 (재접속 X) |
| 매칭 성사 직후 ~ 씬 전환 | 즉시 봇 활성화 | 로컬 재접속 30s 유예 |
| 준비 UI / 로딩 동기화 | 즉시 봇 활성화 (또는 7s 후 강제) | 30s 유예 |
| 시작 데이터 동기화 | 봇 활성화 / 실패 시 로비 | 30s 유예 |
| 인게임 | 즉시 봇 활성화 | 30s → 실패 → 결과 화면 |
| 백그라운드 30s+ 후 복귀 (씬전환/인게임) | (이미 끊겼으면 상대에 봇) | 재접속 초과 → Fail → 결과 |
| 결과 화면 | 무시 (봇 활성화 X) | — |
| 초대 방, 호스트만 대기 | 매칭 대기와 동일 (봇 X) | 매칭 실패 |

**매칭 진입 전 (G 카테고리)** 의 사전 검사 실패는 매칭 시작 자체를 막음 — Photon 연결 안 함.

## 작업 규칙

- 시나리오 ID는 `A1`, `B7` 처럼 카테고리 + 번호. 사용자가 "A6 케이스가 우리 코드에서 처리되나?" 라고 물으면 그 ID에 해당하는 시나리오 항목을 우선 응답.
- args가 카테고리/ID로 주어지면 그 파일만 읽고 응답. **다른 파일은 읽지 말 것** (context 절약).
- 코드 진단 시 추측하지 말 것. 핸들러가 비어 있거나 호출 흐름을 못 찾았으면 `상태: 불확실 / 근거: (없음)`으로 솔직히 표시.
- 권장 조치는 Photon/Quantum 공식 API 명칭을 사용 (`MatchmakingExtensions.JoinRandomRoomAsync`, `QuantumRunner.ShutdownAsync`, `SessionRunner.Arguments` 등). 특정 프로젝트의 클래스명을 가정하지 말 것.
- 진단 결과는 항상 표 + 권장 조치 + 다음 단계 3섹션 구조 유지.
- 진단 결과 보고 후 코드 수정을 자동으로 시작하지 말 것 — 사용자의 명시 승인을 받고 다음 동작 결정.

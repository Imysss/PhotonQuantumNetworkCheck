# quantum-network-skill

Photon Quantum 3 멀티플레이 네트워크 문제 진단 / 시나리오 카탈로그를 담은 Claude Code 스킬.

## 무엇을 하는 스킬인가

Photon Realtime + Quantum 3 기반 멀티플레이 클라이언트를 만들 때 흔히 마주치는 네트워크 케이스(매칭 중 백그라운드, 인게임 끊김, Desync, 봇 fallback, 매칭 전 사전 검사, UX gap 등)를 정리한 카탈로그 + Claude가 코드를 읽고 누락된 핸들링을 표로 정리해주는 진단 절차.

명시 호출 전용. 키워드 자동 트리거 없음.

## 두 가지 모드

### 1. 진단 모드 (Diagnose)
사용자가 "이 매칭 코드 진단해줘" / "끊김 핸들링 빠진 거 찾아줘" 같이 요청하면, Claude가 지정 범위의 코드를 읽고 시나리오별로 구현 상태를 점검 → 우선순위 표로 출력.

```
| ID | 시나리오 | 상태 | 심각도 | 근거 (파일:라인) | 권장 조치 |
|----|----------|------|--------|-------------------|-----------|
| A6 | 매칭 중 백그라운드 진입 | 미구현 | 高 | (없음) | OnApplicationPause에서 Disconnect 호출 |
```

### 2. 체크리스트 / 레퍼런스 모드 (Reference)
"매칭 중 백그라운드 가면 어떻게 처리해야 해?" 같은 질문에 시나리오 카탈로그 항목을 발생 조건 / 증상 / 근본 원인 / 대응 / 우선순위로 정리해 응답.

## 카테고리별 호출 (args 분기)

스킬은 args를 받아 카테고리/시나리오 단위로 응답한다. 다른 영역의 파일은 읽지 않으므로 context 효율적.

```
/photon-quantum-netcheck               # 전체 색인
/photon-quantum-netcheck A             # A 카테고리(매칭)만
/photon-quantum-netcheck B11           # 특정 시나리오만
/photon-quantum-netcheck diag          # 진단 모드
/photon-quantum-netcheck matrix        # 시점×이탈 주체 매트릭스
/photon-quantum-netcheck checklist     # 체크리스트 안내
```

한글 키워드도 인식 (`매칭`, `로딩`, `인게임`, `결과`, `라이프사이클`, `결정론`, `사전검사`, `조용히실패`).

## 시나리오 카탈로그 (8 카테고리, 61개)

각 시나리오는 5필드 구조: **발생 조건 / 증상 / 근본 원인 / 대응 / 우선순위**.

| 카테고리 | 시나리오 수 | 다루는 페이즈 |
|----------|-------------|---------------|
| A. 매칭 (Lobby) | 13 | 매칭 시작 ~ 방 입장 직후 |
| B. 로딩 동기화 (SceneLoading) | 11 | Quantum 세션 시작 ~ 초기 데이터 동기화 |
| C. 인게임 (Gameplay) | 12 | 게임 진행 중 발생하는 네트워크 이슈 |
| D. 결과 (Result) | 4 | 게임 종료 후 보상 / 통계 / 사유 분류 |
| E. 모바일 라이프사이클 | 6 | OS 인터럽트 / 백그라운드 / 종료 |
| F. Quantum 결정론 | 5 | Desync / Snapshot / 결정성 보장 |
| G. 매칭 진입 전 (Pre-match) | 4 | 사전 검사 / 자원 차감 / sanitize / mutex |
| H. UX gap (조용히 실패) | 6 | 이벤트는 발화하지만 UI 없는 케이스 |

## 설치

### 방법 1: Claude Code 마켓플레이스 (권장)

Claude Code 에서 두 줄 실행:

```
/plugin marketplace add Imysss/PhotonQuantumNetworkCheck
/plugin install photon-quantum-netcheck@imysss-plugins
```

설치 후 `/photon-quantum-netcheck` 또는 자연어 ("매칭 진단해줘") 로 호출.

### 방법 2: 수동 복사 (legacy)

마켓플레이스를 안 쓰는 경우 SKILL 파일들을 직접 복사:

```bash
git clone https://github.com/Imysss/PhotonQuantumNetworkCheck.git
cp -r PhotonQuantumNetworkCheck/plugins/photon-quantum-netcheck/skills/photon-quantum-netcheck ~/.claude/skills/   # 사용자 레벨
# 또는 프로젝트 레벨:
# cp -r PhotonQuantumNetworkCheck/plugins/photon-quantum-netcheck/skills/photon-quantum-netcheck <project>/.claude/skills/
```

## 디렉토리 구조

```
PhotonQuantumNetworkCheck/                            # 레포 루트 (= 마켓플레이스)
├── .claude-plugin/
│   └── marketplace.json                              # 마켓플레이스 정의
├── plugins/
│   └── photon-quantum-netcheck/                      # 플러그인 본체
│       ├── .claude-plugin/
│       │   └── plugin.json                           # 플러그인 매니페스트
│       └── skills/
│           └── photon-quantum-netcheck/              # 실제 스킬
│               ├── SKILL.md                          # 진입점 (frontmatter + 카탈로그 색인 + 매트릭스)
│               ├── references/                       # 시나리오 상세 (5필드, 8 카테고리)
│               │   ├── A-matchmaking.md
│               │   ├── B-loading-sync.md
│               │   ├── C-in-game-disconnect.md
│               │   ├── D-result-phase.md
│               │   ├── E-mobile-lifecycle.md
│               │   ├── F-quantum-determinism.md
│               │   ├── G-pre-match.md
│               │   └── H-ux-gap.md
│               ├── checklists/                       # 구현·PR 점검용
│               │   ├── matchmaking.md
│               │   ├── in-game.md
│               │   ├── lifecycle.md
│               │   ├── pre-match.md
│               │   └── ux-gap.md
│               └── diagnostics/
│                   └── procedure.md                  # 진단 모드 절차서
└── README.md
```

## 호환성

- Photon Quantum 3 (v3.0.x 검증)
- Photon Realtime 5.x
- Unity 2022.3 LTS 기준 (다른 버전도 대부분 호환)

특정 프로젝트의 클래스명을 가정하지 않음. Photon/Quantum 공식 API 명칭(`SessionRunner.Arguments`, `QuantumRunner`, `LoadBalancingClient`, `OnPlayerLeftRoom` 등) 기준으로 작성.

### 게임 정책 가정

이 스킬은 다음을 기본값으로 시나리오를 기술한다 — 다른 정책을 채택한 게임은 해석해서 적용 (SKILL.md "게임 정책 가정" 섹션 참조):

- **대체 세션 정책**: 상대 이탈 시 봇/AI로 슬롯 인계 (모바일 캐주얼 게임 흔한 정책). 무승부/종료/빈 슬롯 진행 정책도 지원.
- **재접속 grace**: 30초 (Photon `KeepAliveInBackground` 와 동일).
- **매칭 인원**: 2인 1:1 또는 2인 협동 기본. 4인 이상은 인원/슬롯/Late Join 정책 재해석.
- **모바일 라이프사이클**: Android/iOS 한정. PC/콘솔은 E 카테고리 무용.

### 게임 유형별 적용성

| 게임 유형 | 적용성 |
|-----------|--------|
| 2인 PvP / 협동 모바일 게임 | 90% — 거의 그대로 |
| 4인 협동 모바일 게임 | 70~80% — 인원·슬롯 분기 재해석 |
| 팀전 PvP (3v3 이상) | 50~60% — 대체 정책 / 결과 분류 재해석 |
| PC / 콘솔 Quantum 게임 | 40~50% — E(라이프사이클), G(자원) 거의 skip |

## 기여

시나리오 추가 / 권장 조치 개선 PR 환영. 시나리오는 5필드 구조 유지:
- 발생 조건
- 증상
- 근본 원인
- 대응
- 우선순위 (高/中/低)

## 라이센스

MIT (별도 명시 시 변경)

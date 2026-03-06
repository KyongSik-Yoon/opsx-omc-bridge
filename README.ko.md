# opsx-omc-bridge

[English](README.md)

**OpenSpec** 기획 산출물을 **Oh My ClaudeCode (OMC)** 실행 모드로 자동 위임하는 Claude Code 플러그인. 스펙 준비와 구현 사이의 수동 핸드오프를 제거한다.

## 문제

OpenSpec으로 스펙(`proposal.md`, `design.md`, `tasks.md`)을 준비한 뒤, OMC로 구현을 위임할 때 반복적인 수동 작업이 필요하다:

1. `tasks.md`를 읽고 미완료 항목을 추출
2. `design.md`의 제약사항 정리
3. 적절한 OMC 모드 선택
4. 컨텍스트를 조합한 프롬프트 작성

이 플러그인이 전체 핸드오프 과정을 자동화한다.

## 스킬 구성

| 스킬 | 대상 | OMC 모드 | 트리거 예시 |
|------|------|----------|------------|
| **quick** | 태스크 3개 이하, 단일 도메인 | `autopilot` | "빠르게 구현해", "바로 적용해" |
| **deploy** | 태스크 4개 이상, 다중 Phase/도메인 | `team` 또는 `ralph` (자동 판단) | "팀으로 구현해", "ralph로 넘겨" |

### quick

간단한 변경에 적합. OpenSpec 산출물을 읽어 컨텍스트를 조립하고, OMC `autopilot`에 위임하여 순차 실행한다.

### deploy

복잡한 변경에 적합. 태스크 구조(Phase 수, 의존성, 도메인 분포)를 분석하여 최적 실행 전략을 자동 결정한다:

| 조건 | OMC 모드 | 이유 |
|------|----------|------|
| Phase가 순차적이고 의존성 강함 | `ralph` | 한 단계씩 검증하며 진행 |
| Phase 내 태스크가 독립적 | `team N:executor` | 병렬 처리 가능 |
| 단일 Phase, 태스크 4+ | `team N:executor` | 단순 병렬화 |
| 리팩토링/성능 개선 (검증 반복 필요) | `ralph` | Architect 검증까지 반복 |
| Phase 순차 + Phase 내 병렬 가능 | `ralph` + Phase별 `team` | 하이브리드 |

## 워크플로우

```
/opsx:propose "기능 설명"     ← OpenSpec으로 스펙 수립
        ↓
/opsx:ff                      ← 산출물 일괄 생성
        ↓
"빠르게 구현해"               ← quick 자동 트리거 → autopilot
또는
"팀으로 구현해"               ← deploy 자동 트리거 → team/ralph
        ↓
/opsx:verify                  ← 스펙 대비 검증
        ↓
/opsx:archive                 ← 아카이빙
```

## 설치

```bash
cd /path/to/opsx-omc-bridge
claude /plugin install --plugin-dir .
```

## 전제 조건

- **OpenSpec**이 대상 프로젝트에 초기화되어 있을 것 (`openspec/` 디렉토리)
- **Oh My ClaudeCode** 플러그인이 설치되어 있을 것
- **tmux** 세션 활성화 (deploy의 `team` 모드에 필요)

## 디렉토리 구조

```
opsx-omc-bridge/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── quick/
│   │   └── SKILL.md
│   └── deploy/
│       └── SKILL.md
├── README.md
└── README.ko.md
```

## 라이선스

MIT

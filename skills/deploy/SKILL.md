---
name: deploy
description: OpenSpec에서 준비된 복잡한 변경사항을 Oh My ClaudeCode(OMC)의 team/ralph 모드로 위임하는 브릿지 스킬. 사용자가 "팀으로 구현해", "병렬로 돌려", "ralph로 넘겨", "큰 작업 시작", "전체 구현 위임" 같은 표현을 사용하거나, OpenSpec change에 4개 이상의 태스크, 다중 도메인, 또는 Phase가 나뉜 복잡한 작업이 있을 때 트리거한다. 병렬화가 가능한 대규모 구현, 완료까지 지속적 검증이 필요한 리팩토링에 적합하다.
---

# OpenSpec → OMC Deploy Bridge

OpenSpec의 planning 산출물이 준비된 복잡한 작업을 OMC의 team 또는 ralph 모드로 위임하는 브릿지 스킬이다. 태스크 분석을 통해 최적의 OMC 실행 전략을 자동으로 결정한다.

## 언제 이 스킬을 사용하는가

- tasks.md에 4개 이상의 태스크가 있을 때
- 여러 Phase로 나뉜 작업일 때
- 다중 도메인(프론트엔드 + 백엔드, DDL + 서비스 + 테스트 등)에 걸친 변경일 때
- 완료까지 반복적 검증이 필요한 리팩토링일 때

## 실행 흐름

### Step 1: 대상 change 식별

`quick`과 동일하게 active change를 스캔한다.

```bash
ls -d openspec/changes/*/tasks.md 2>/dev/null | grep -v archive
```

여러 change가 있으면 사용자에게 선택을 요청한다.

### Step 2: 전체 산출물 검증

복잡한 작업에는 모든 산출물이 필요하다:

```bash
CHANGE_DIR="openspec/changes/<change-name>"
MISSING=""
test -f "$CHANGE_DIR/proposal.md" || MISSING="$MISSING proposal.md"
test -f "$CHANGE_DIR/design.md"   || MISSING="$MISSING design.md"
test -f "$CHANGE_DIR/tasks.md"    || MISSING="$MISSING tasks.md"
# delta specs 존재 확인
ls "$CHANGE_DIR/specs/" 2>/dev/null | head -1 || MISSING="$MISSING delta-specs"
```

- tasks.md 또는 design.md가 없으면 중단하고 `/opsx:continue` 또는 `/opsx:ff`를 안내
- delta specs가 없으면 경고하되 진행 (모든 change에 delta가 필요한 건 아님)

### Step 3: 태스크 분석 및 실행 전략 결정

tasks.md를 파싱해서 작업의 구조를 분석한다:

**분석 항목:**
- 총 태스크 수 (미완료 `- [ ]` 기준)
- Phase 수 (## Phase, ## 단계 등의 헤더)
- 태스크 간 의존성 (Phase N의 태스크가 Phase N-1 결과에 의존하는지)
- 도메인 분포 (DDL, Kotlin 서비스, 테스트, 프론트엔드 등)

**전략 결정 매트릭스:**

| 조건 | OMC 모드 | 이유 |
|------|----------|------|
| Phase가 순차적이고 의존성 강함 | `ralph` | 한 단계씩 검증하며 진행 |
| Phase 내 태스크가 독립적 | `team N:executor` | 병렬 처리 가능 |
| 단일 Phase, 태스크 4+ | `team N:executor` | 단순 병렬화 |
| 리팩토링/성능 개선 (검증 반복 필요) | `ralph` | Architect 검증까지 반복 |
| Phase 순차 + Phase 내 병렬 가능 | `ralph` + Phase별 `team` | 하이브리드 |

에이전트 수 결정:
- 독립 태스크 수의 절반 (반올림), 최소 2, 최대 5
- 예: 독립 태스크 6개 → `team 3:executor`

### Step 4: 풍부한 컨텍스트 조립

모든 산출물에서 컨텍스트를 추출한다:

1. **proposal.md** → 동기, 범위, 제약조건
2. **delta specs** → GIVEN/WHEN/THEN 시나리오 (검증 기준으로 활용)
3. **design.md** → 아키텍처 결정, 컴포넌트 설계, 데이터 흐름
4. **tasks.md** → 전체 체크리스트

openspec/project.md도 읽어서 프로젝트 전역 컨텍스트(기술 스택, 컨벤션)를 포함한다.

### Step 5: 실행 전략별 위임

#### 전략 A: ralph 모드 (순차 + 검증 반복)

```
ralph 다음 변경사항을 구현해줘. 완료될 때까지 멈추지 마.

## 프로젝트 컨텍스트
[project.md에서 추출한 기술 스택, 컨벤션]

## 변경 배경
[proposal.md 요약]

## 아키텍처 결정 (반드시 준수)
[design.md 핵심 내용]

## 검증 시나리오
[delta specs의 GIVEN/WHEN/THEN - Architect가 검증할 기준]

## 태스크 (순서대로 진행)
[tasks.md 전문]

## 지침
- 각 Phase 완료 시 해당 Phase의 GIVEN/WHEN/THEN 시나리오로 검증
- design.md의 아키텍처 결정을 절대 벗어나지 말 것
- 각 태스크 완료 시 tasks.md를 [x]로 업데이트
- 기존 테스트가 깨지면 즉시 수정
- 모든 Phase 완료 후 전체 테스트 스위트 실행
```

#### 전략 B: team 모드 (병렬 실행)

```
team [N]:executor 다음 작업을 병렬로 구현해줘.

## 프로젝트 컨텍스트
[project.md에서 추출]

## 변경 배경
[proposal.md 요약]

## 아키텍처 결정 (모든 에이전트가 준수)
[design.md 핵심 내용]

## 태스크 분배
### 에이전트 1: [도메인/역할]
[해당 태스크 목록]

### 에이전트 2: [도메인/역할]
[해당 태스크 목록]

### 에이전트 N: [도메인/역할]
[해당 태스크 목록]

## 공통 지침
- 각자 완료한 태스크를 tasks.md에서 [x]로 체크
- design.md의 아키텍처 결정을 반드시 따를 것
- 다른 에이전트 담당 파일을 수정하지 말 것
- 완료 후 각자 담당 영역 테스트 실행
```

#### 전략 C: 하이브리드 (Phase 순차, Phase 내 병렬)

Phase를 순서대로 진행하되, 각 Phase 안에서는 team 모드를 사용한다.
이 경우 사용자에게 전략을 설명하고, Phase별로 실행할지 전체를 ralph에게 맡길지 선택하게 한다.

### Step 6: 실행 전 사용자 확인

위임하기 전에 분석 결과를 사용자에게 보여주고 확인을 받는다:

```
📋 OpenSpec → OMC 위임 분석

Change: add-jmx-metrics-table
태스크: 8개 (3 Phase)
전략: ralph (Phase 간 의존성 있음)

Phase 1: DDL & Infrastructure (3 태스크)
Phase 2: Service Layer (3 태스크) — Phase 1 결과에 의존
Phase 3: Testing (2 태스크) — Phase 2 결과에 의존

검증 시나리오: 3개 (delta specs 기반)

이대로 진행할까요? [Y/n]
전략을 변경하려면 "team으로 바꿔" 또는 "autopilot으로 해"라고 말해주세요.
```

### Step 7: 완료 후 안내

OMC 실행이 완료되면:

1. tasks.md의 체크 상태 확인 — 미완료 항목이 있으면 보고
2. 다음 단계 안내:
   - `/opsx:verify` — delta specs의 시나리오 기반으로 구현 검증
   - `/opsx:archive` — 완료 확인 후 아카이빙
   - "추가 수정이 필요하면 tasks.md에 태스크를 추가하고 다시 이 스킬을 사용하세요"

## 주의사항

- team 모드는 tmux 세션이 활성화되어 있어야 한다. Termux 환경에서는 tmux 설정 상태를 먼저 확인한다.
- 에이전트 수를 너무 많이 잡으면 rate limit에 걸릴 수 있다. 5개 이하를 권장한다.
- delta specs가 없는 change에서는 검증 시나리오를 tasks.md의 항목 자체로 대체한다.
- 사용자가 전략에 동의하지 않으면 즉시 조정한다. 이 스킬의 판단은 제안이지 강제가 아니다.

---
name: quick
description: OpenSpec에서 준비된 간단한 변경사항을 Oh My ClaudeCode(OMC)의 autopilot으로 바로 위임하는 브릿지 스킬. 사용자가 "빠르게 구현해", "이거 바로 적용해", "autopilot으로 넘겨", "opsx 적용", "스펙 구현 시작" 같은 표현을 사용하거나, OpenSpec change 폴더에 tasks.md가 준비되어 있고 단순 구현만 남은 상황에서 트리거한다. 3개 이하의 태스크, 단일 도메인, 병렬화가 불필요한 작업에 적합하다.
---

# OpenSpec → OMC Quick Bridge

OpenSpec의 planning 산출물이 준비된 간단한 작업을 OMC autopilot으로 넘기는 브릿지 스킬이다.
수동으로 tasks.md를 복사-붙이기하거나 컨텍스트를 재설명하는 과정을 제거하는 것이 목적이다.

## 언제 이 스킬을 사용하는가

- OpenSpec change 폴더에 tasks.md가 존재하고 태스크가 3개 이하일 때
- 단일 도메인 변경 (파일 수정 범위가 좁음)
- 병렬 에이전트가 필요 없는 직선적 작업

## 실행 흐름

### Step 1: 대상 change 식별

`openspec/changes/` 디렉토리를 스캔해서 active change 목록을 확인한다.

```bash
ls -d openspec/changes/*/tasks.md 2>/dev/null | grep -v archive
```

- change가 1개면 자동 선택
- 여러 개면 사용자에게 어떤 change를 구현할지 확인
- 없으면 "먼저 /opsx:propose 또는 /opsx:ff로 스펙을 준비하세요"라고 안내하고 중단

### Step 2: 산출물 검증

선택된 change 폴더에서 최소 필요 파일을 확인한다:

```bash
CHANGE_DIR="openspec/changes/<change-name>"
# 필수: tasks.md
test -f "$CHANGE_DIR/tasks.md" || echo "MISSING: tasks.md"
# 권장: proposal.md (컨텍스트용)
test -f "$CHANGE_DIR/proposal.md" || echo "WARNING: proposal.md 없음 - 컨텍스트가 부족할 수 있음"
```

tasks.md가 없으면 중단한다. proposal.md가 없으면 경고만 하고 진행한다.

### Step 3: 컨텍스트 수집

다음 순서로 파일을 읽어서 구현 컨텍스트를 조립한다:

1. `proposal.md` — 변경의 동기와 범위 (Why/What)
2. `design.md` — 기술적 접근 방식 (How) — 있으면 읽고, 없으면 스킵
3. `tasks.md` — 구현 체크리스트 (전문 읽기)

읽은 내용에서 다음을 추출한다:
- **목표 요약**: proposal.md의 Intent/Why 섹션에서 1-2문장
- **기술 제약**: design.md의 주요 결정사항 (엔진 선택, 패턴 등)
- **미완료 태스크**: tasks.md에서 `- [ ]`인 항목만 필터링

### Step 4: OMC autopilot 위임

수집한 컨텍스트를 하나의 autopilot 지시로 조합한다. 핵심 원칙:

- tasks.md의 원문을 직접 참조시킨다 (요약하지 않음)
- design.md의 제약사항을 명시적으로 전달한다
- 체크리스트 형태를 유지해서 OMC가 진행 상황을 추적할 수 있게 한다

구성할 프롬프트 구조:

```
autopilot 다음 작업을 구현해줘.

## 배경
[proposal.md에서 추출한 1-2문장 요약]

## 기술 제약
[design.md에서 추출한 핵심 결정사항. 없으면 이 섹션 생략]

## 태스크
[tasks.md의 미완료 항목 전문]

## 지침
- 각 태스크 완료 시 tasks.md의 해당 항목을 [x]로 체크할 것
- design.md의 아키텍처 결정을 반드시 따를 것
- 기존 테스트가 있으면 깨지지 않는지 확인할 것
```

### Step 5: 완료 후 안내

OMC autopilot이 완료되면 사용자에게 다음 단계를 안내한다:

- "구현이 완료되었습니다. `/opsx:verify`로 스펙 대비 검증하거나, `/opsx:archive`로 아카이빙할 수 있습니다."

## 주의사항

- 이 스킬은 OpenSpec의 `/opsx:apply`를 대체하는 것이 아니라, OMC의 자율 실행 능력을 활용하는 대안 경로이다.
- tasks.md에 4개 이상의 태스크가 있거나, 여러 도메인에 걸친 변경이면 `deploy` 스킬 사용을 권장한다.
- OMC가 설치되어 있지 않으면 이 스킬은 동작하지 않는다. 일반 `/opsx:apply`를 안내한다.

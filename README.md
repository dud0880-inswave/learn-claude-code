# Learn Claude Code - Bash만 있으면 에이전트를 만들 수 있습니다

[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![Tests](https://github.com/shareAI-lab/learn-claude-code/actions/workflows/test.yml/badge.svg)](https://github.com/shareAI-lab/learn-claude-code/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)

> **면책조항**: 이 프로젝트는 [shareAI Lab](https://github.com/shareAI-lab)의 독립적인 교육 프로젝트입니다. Anthropic과 제휴, 보증 또는 후원 관계가 없습니다. "Claude Code"는 Anthropic의 상표입니다.

**AI 에이전트를 직접 만들어보며 현대 AI 에이전트의 작동 원리를 배워보세요.**

[English](./README_en.md)

---

## 왜 이 저장소를 만들었나요?

우리는 Claude Code에 대한 존경심으로 이 저장소를 만들었습니다 - **세계에서 가장 뛰어난 AI 코딩 에이전트라고 믿습니다**. 처음에는 행동 관찰과 추측을 통해 설계를 리버스 엔지니어링하려고 시도했습니다. 그때 발표한 분석은 부정확성, 근거 없는 추측, 기술적 오류로 가득했습니다. Claude Code 팀과 그 콘텐츠로 인해 잘못된 정보를 접한 모든 분들께 깊이 사과드립니다.

지난 6개월 동안 실제 에이전트 시스템을 구축하고 반복하면서 **"진정한 AI 에이전트란 무엇인가"**에 대한 우리의 이해가 근본적으로 바뀌었습니다. 이러한 통찰을 여러분과 공유하고자 합니다. 이전의 모든 추측성 콘텐츠는 삭제되고 독창적인 교육 자료로 대체되었습니다.

---

> **[Kode CLI](https://github.com/shareAI-lab/Kode)**, **Claude Code**, **Cursor** 및 [Agent Skills Spec](https://agentskills.io/specification)을 지원하는 모든 에이전트와 함께 작동합니다.

<img height="400" alt="demo" src="https://github.com/user-attachments/assets/0e1e31f8-064f-4908-92ce-121e2eb8d453" />

## 배우게 될 내용

이 튜토리얼을 완료하면 다음을 이해하게 됩니다:

- **에이전트 루프** - 모든 AI 코딩 에이전트의 놀랍도록 간단한 패턴
- **도구 설계** - AI 모델이 현실 세계와 상호작용할 수 있게 하는 방법
- **명시적 계획** - 제약 조건을 사용하여 AI 행동을 예측 가능하게 만들기
- **컨텍스트 관리** - 서브에이전트 격리를 통해 에이전트 메모리를 깔끔하게 유지
- **지식 주입** - 재학습 없이 필요에 따라 도메인 전문 지식 로딩

## 학습 경로

```
여기서 시작
    |
    v
[v0: Bash Agent] -----> "도구 하나면 충분합니다"
    |                    16-50줄
    v
[v1: Basic Agent] ----> "완전한 에이전트 패턴"
    |                    도구 4개, ~200줄
    v
[v2: Todo Agent] -----> "계획을 명시적으로"
    |                    +TodoManager, ~300줄
    v
[v3: Subagent] -------> "분할 정복"
    |                    +Task 도구, ~450줄
    v
[v4: Skills Agent] ---> "필요에 따른 도메인 전문성"
                         +Skill 도구, ~550줄
```

**권장 접근 방식:**
1. v0을 먼저 읽고 실행 - 핵심 루프 이해
2. v0과 v1 비교 - 도구가 어떻게 발전하는지 확인
3. 계획 패턴을 위해 v2 학습
4. 복잡한 작업 분해를 위해 v3 탐색
5. 확장 가능한 에이전트 구축을 위해 v4 마스터

## 빠른 시작

```bash
# 저장소 클론
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code

# 의존성 설치
pip install -r requirements.txt

# API 키 설정
cp .env.example .env
# .env 파일에 ANTHROPIC_API_KEY를 입력하세요

# 원하는 버전 실행
python v0_bash_agent.py      # 최소 버전 (여기서 시작!)
python v1_basic_agent.py     # 핵심 에이전트 루프
python v2_todo_agent.py      # + Todo 계획
python v3_subagent.py        # + 서브에이전트
python v4_skills_agent.py    # + 스킬
```

## 핵심 패턴

모든 코딩 에이전트는 이 루프일 뿐입니다:

```python
while True:
    response = model(messages, tools)
    if response.stop_reason != "tool_use":
        return response.text
    results = execute(response.tool_calls)
    messages.append(results)
```

이게 전부입니다. 모델은 완료될 때까지 도구를 호출합니다. 나머지는 모두 개선 사항입니다.

## 버전 비교

| 버전 | 줄 수 | 도구 | 핵심 추가 사항 | 주요 인사이트 |
|------|-------|------|----------------|---------------|
| [v0](./v0_bash_agent.py) | ~50 | bash | 재귀적 서브에이전트 | 도구 하나면 충분 |
| [v1](./v1_basic_agent.py) | ~200 | bash, read, write, edit | 핵심 루프 | 모델이 곧 에이전트 |
| [v2](./v2_todo_agent.py) | ~300 | +TodoWrite | 명시적 계획 | 제약이 복잡성을 가능하게 함 |
| [v3](./v3_subagent.py) | ~450 | +Task | 컨텍스트 격리 | 깨끗한 컨텍스트 = 더 나은 결과 |
| [v4](./v4_skills_agent.py) | ~550 | +Skill | 지식 로딩 | 재학습 없는 전문성 |

## 파일 구조

```
learn-claude-code/
├── v0_bash_agent.py       # ~50줄: 도구 1개, 재귀적 서브에이전트
├── v0_bash_agent_mini.py  # ~16줄: 극단적 압축
├── v1_basic_agent.py      # ~200줄: 도구 4개, 핵심 루프
├── v2_todo_agent.py       # ~300줄: + TodoManager
├── v3_subagent.py         # ~450줄: + Task 도구, 에이전트 레지스트리
├── v4_skills_agent.py     # ~550줄: + Skill 도구, SkillLoader
├── skills/                # 예제 스킬 (pdf, code-review, mcp-builder, agent-builder)
├── docs/                  # 기술 문서 (EN + ZH + JA)
├── articles/              # 블로그 스타일 글 (ZH)
└── tests/                 # 유닛 및 통합 테스트
```

## 문서

### 기술 튜토리얼 (docs/)

- [v0: Bash만 있으면 됩니다](./docs/v0-bash-is-all-you-need_kr.md)
- [v1: 모델이 곧 에이전트](./docs/v1-model-as-agent_kr.md)
- [v2: 구조화된 계획](./docs/v2-structured-planning_kr.md)
- [v3: 서브에이전트 메커니즘](./docs/v3-subagent-mechanism_kr.md)
- [v4: 스킬 메커니즘](./docs/v4-skills-mechanism_kr.md)

### 아티클

블로그 스타일 설명은 [articles/](./articles/)를 참조하세요.

## 스킬 시스템 사용하기

### 포함된 예제 스킬

| 스킬 | 용도 |
|------|------|
| [agent-builder](./skills/agent-builder/) | 메타 스킬: 에이전트 구축 방법 |
| [code-review](./skills/code-review/) | 체계적인 코드 리뷰 방법론 |
| [pdf](./skills/pdf/) | PDF 조작 패턴 |
| [mcp-builder](./skills/mcp-builder/) | MCP 서버 개발 |

### 새 에이전트 스캐폴드

```bash
# agent-builder 스킬을 사용하여 새 프로젝트 생성
python skills/agent-builder/scripts/init_agent.py my-agent

# 복잡도 수준 지정
python skills/agent-builder/scripts/init_agent.py my-agent --level 0  # 최소
python skills/agent-builder/scripts/init_agent.py my-agent --level 1  # 도구 4개
```

### 프로덕션용 스킬 설치

```bash
# Kode CLI (권장)
kode plugins install https://github.com/shareAI-lab/shareAI-skills

# Claude Code
claude plugins install https://github.com/shareAI-lab/shareAI-skills
```

## 설정

```bash
# .env 파일 옵션
ANTHROPIC_API_KEY=sk-ant-xxx      # 필수: API 키
ANTHROPIC_BASE_URL=https://...    # 선택: API 프록시용
MODEL_ID=claude-sonnet-4-5-20250929  # 선택: 모델 선택
```

## 관련 프로젝트

| 저장소 | 설명 |
|--------|------|
| [Kode](https://github.com/shareAI-lab/Kode) | 프로덕션급 오픈소스 에이전트 CLI |
| [shareAI-skills](https://github.com/shareAI-lab/shareAI-skills) | 프로덕션 스킬 컬렉션 |
| [Agent Skills Spec](https://agentskills.io/specification) | 공식 스펙 |

## 철학

> **모델이 80%. 코드가 20%.**

Kode나 Claude Code 같은 현대 에이전트가 작동하는 이유는 영리한 엔지니어링 때문이 아니라 모델이 에이전트로 훈련되었기 때문입니다. 우리의 역할은 도구를 제공하고 방해하지 않는 것입니다.

## 기여하기

기여를 환영합니다! 이슈와 풀 리퀘스트를 자유롭게 제출해 주세요.

- `skills/`에 새로운 예제 스킬 추가
- `docs/`의 문서 개선
- [Issues](https://github.com/shareAI-lab/learn-claude-code/issues)를 통해 버그 보고 또는 기능 제안

## 라이선스

MIT

---

**모델이 곧 에이전트. 이것이 전부입니다.**

[@baicai003](https://x.com/baicai003) | [shareAI Lab](https://github.com/shareAI-lab)

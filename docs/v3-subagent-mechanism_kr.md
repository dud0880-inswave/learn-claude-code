# v3: 서브에이전트 메커니즘

**~450줄. +1 도구. 분할 정복.**

v2는 계획을 추가합니다. 하지만 "코드베이스를 탐색한 다음 인증을 리팩토링해라"와 같은 대규모 작업의 경우, 단일 에이전트는 컨텍스트 한계에 도달합니다. 탐색이 20개 파일을 히스토리에 덤프합니다. 리팩토링은 집중력을 잃습니다.

v3은 **Task 도구**를 추가합니다: 격리된 컨텍스트를 가진 자식 에이전트를 생성합니다.

## 문제점

단일 에이전트 컨텍스트 오염:

```
Main Agent 히스토리:
  [탐색 중...] cat file1.py -> 500줄
  [탐색 중...] cat file2.py -> 300줄
  ... 15개 더 많은 파일 ...
  [이제 리팩토링...] "잠깐, file1에 뭐가 있었지?"
```

해결책: **탐색을 서브에이전트에게 위임**:

```
Main Agent 히스토리:
  [Task: 코드베이스 탐색]
    -> 서브에이전트가 20개 파일 탐색
    -> 반환: "인증은 src/auth/에, DB는 src/models/에"
  [이제 깨끗한 컨텍스트로 리팩토링]
```

## 에이전트 타입 레지스트리

각 에이전트 타입은 기능을 정의합니다:

```python
AGENT_TYPES = {
    "explore": {
        "description": "검색과 분석을 위한 읽기 전용",
        "tools": ["bash", "read_file"],  # 쓰기 없음
        "prompt": "검색하고 분석하세요. 절대 수정하지 마세요. 간결한 요약을 반환하세요."
    },
    "code": {
        "description": "구현을 위한 전체 에이전트",
        "tools": "*",  # 모든 도구
        "prompt": "효율적으로 변경사항을 구현하세요."
    },
    "plan": {
        "description": "계획과 분석",
        "tools": ["bash", "read_file"],  # 읽기 전용
        "prompt": "분석하고 번호가 매겨진 계획을 출력하세요. 파일을 변경하지 마세요."
    }
}
```

## Task 도구

```python
{
    "name": "Task",
    "description": "집중된 하위 작업을 위한 서브에이전트 생성",
    "input_schema": {
        "description": "짧은 작업명 (3-5 단어)",
        "prompt": "상세한 지시",
        "agent_type": "explore | code | plan"
    }
}
```

메인 에이전트가 Task 호출 → 자식 에이전트 실행 → 요약 반환.

## 서브에이전트 실행

Task 도구의 핵심:

```python
def run_task(description, prompt, agent_type):
    config = AGENT_TYPES[agent_type]

    # 1. 에이전트별 시스템 프롬프트
    sub_system = f"You are a {agent_type} subagent.\n{config['prompt']}"

    # 2. 필터링된 도구
    sub_tools = get_tools_for_agent(agent_type)

    # 3. 격리된 히스토리 (핵심: 부모 컨텍스트 없음)
    sub_messages = [{"role": "user", "content": prompt}]

    # 4. 같은 쿼리 루프
    while True:
        response = client.messages.create(
            model=MODEL, system=sub_system,
            messages=sub_messages, tools=sub_tools
        )
        if response.stop_reason != "tool_use":
            break
        # 도구 실행, 결과 추가...

    # 5. 최종 텍스트만 반환
    return extract_final_text(response)
```

**핵심 개념:**

| 개념 | 구현 |
|------|------|
| 컨텍스트 격리 | 새로운 `sub_messages = []` |
| 도구 필터링 | `get_tools_for_agent()` |
| 특화된 동작 | 에이전트별 시스템 프롬프트 |
| 결과 추상화 | 최종 텍스트만 반환 |

## 도구 필터링

```python
def get_tools_for_agent(agent_type):
    allowed = AGENT_TYPES[agent_type]["tools"]
    if allowed == "*":
        return BASE_TOOLS  # Task 없음 (데모에서 재귀 없음)
    return [t for t in BASE_TOOLS if t["name"] in allowed]
```

- `explore`: bash + read_file만
- `code`: 모든 도구
- `plan`: bash + read_file만

서브에이전트는 Task 도구를 받지 않습니다 (이 데모에서 무한 재귀 방지).

## 진행 상황 표시

서브에이전트 출력은 메인 채팅을 오염시키지 않습니다:

```
You: 코드베이스 탐색해줘
> Task: 코드베이스 탐색
  [explore] 코드베이스 탐색 ... 도구 5개, 3.2초
  [explore] 코드베이스 탐색 - 완료 (도구 8개, 5.1초)

제가 발견한 것은: ...
```

실시간 진행 상황, 깔끔한 최종 출력.

## 일반적인 흐름

```
사용자: "인증을 JWT로 리팩토링해줘"

메인 에이전트:
  1. Task(explore): "모든 인증 관련 파일 찾기"
     -> 서브에이전트가 10개 파일 읽음
     -> 반환: "인증은 src/auth/login.py에, 세션은..."

  2. Task(plan): "JWT 마이그레이션 설계"
     -> 서브에이전트가 구조 분석
     -> 반환: "1. jwt 라이브러리 추가 2. 토큰 유틸 생성..."

  3. Task(code): "JWT 토큰 구현"
     -> 서브에이전트가 코드 작성
     -> 반환: "jwt_utils.py 생성, login.py 업데이트"

  4. 변경사항 요약
```

각 서브에이전트는 깨끗한 컨텍스트를 가집니다. 메인 에이전트는 집중을 유지합니다.

## 비교

| 측면 | v2 | v3 |
|------|----|----|
| 컨텍스트 | 단일, 성장 | 작업별 격리 |
| 탐색 | 히스토리 오염 | 서브에이전트에 포함 |
| 병렬성 | 없음 | 가능 (데모에는 없음) |
| 추가된 코드 | ~300줄 | ~450줄 |

## 패턴

```
복잡한 작업
  └─ 메인 에이전트 (코디네이터)
       ├─ 서브에이전트 A (explore) -> 요약
       ├─ 서브에이전트 B (plan) -> 계획
       └─ 서브에이전트 C (code) -> 결과
```

같은 에이전트 루프, 다른 컨텍스트. 이것이 전체 비결입니다.

---

**분할 정복. 컨텍스트 격리.**

[← v2](./v2-structured-planning_kr.md) | [README로 돌아가기](../README.md) | [v4 →](./v4-skills-mechanism_kr.md)

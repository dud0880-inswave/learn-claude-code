# v1: 모델이 곧 에이전트

**~200줄. 도구 4개. 모든 코딩 에이전트의 본질.**

Claude Code의 비밀은? **비밀 같은 건 없습니다.**

CLI 꾸미기, 진행 바, 권한 시스템을 걷어내면 놀랍도록 단순한 것이 남습니다: 모델이 작업이 완료될 때까지 도구를 호출하게 하는 루프.

## 핵심 인사이트

전통적인 어시스턴트:
```
사용자 -> 모델 -> 텍스트 응답
```

에이전트 시스템:
```
사용자 -> 모델 -> [도구 -> 결과]* -> 응답
                      ^___________|
```

별표(*)가 중요합니다. 모델은 작업이 완료되었다고 판단할 때까지 **반복적으로** 도구를 호출합니다. 이것이 챗봇을 자율적인 에이전트로 변환합니다.

**핵심 인사이트**: 모델이 의사결정자입니다. 코드는 그저 도구를 제공하고 루프를 실행할 뿐입니다.

## 4가지 필수 도구

Claude Code에는 ~20개의 도구가 있습니다. 하지만 4개가 90%의 사용 사례를 커버합니다:

| 도구 | 목적 | 예시 |
|------|------|------|
| `bash` | 명령 실행 | `npm install`, `git status` |
| `read_file` | 내용 읽기 | `src/index.ts` 보기 |
| `write_file` | 생성/덮어쓰기 | `README.md` 생성 |
| `edit_file` | 정밀한 변경 | 함수 교체 |

이 4가지 도구로 모델은 다음을 할 수 있습니다:
- 코드베이스 탐색 (`bash: find, grep, ls`)
- 코드 이해 (`read_file`)
- 변경 작업 (`write_file`, `edit_file`)
- 무엇이든 실행 (`bash: python, npm, make`)

## 에이전트 루프

전체 에이전트를 하나의 함수로:

```python
def agent_loop(messages):
    while True:
        # 1. 모델에게 질문
        response = client.messages.create(
            model=MODEL, system=SYSTEM,
            messages=messages, tools=TOOLS
        )

        # 2. 텍스트 출력 인쇄
        for block in response.content:
            if hasattr(block, "text"):
                print(block.text)

        # 3. 도구 호출이 없으면 완료
        if response.stop_reason != "tool_use":
            return messages

        # 4. 도구 실행, 계속
        results = []
        for tc in response.tool_calls:
            output = execute_tool(tc.name, tc.input)
            results.append({"type": "tool_result", "tool_use_id": tc.id, "content": output})

        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": results})
```

**왜 이것이 작동하는가:**
1. 모델이 루프를 제어 (`stop_reason != "tool_use"`일 때까지 도구 계속 호출)
2. 결과가 컨텍스트가 됨 ("user" 메시지로 다시 공급)
3. 메모리는 자동 (messages 리스트가 히스토리 누적)

## 시스템 프롬프트

필요한 유일한 "설정":

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.

Loop: think briefly -> use tools -> report results.

Rules:
- Prefer tools over prose. Act, don't just explain.
- Never invent file paths. Use ls/find first if unsure.
- Make minimal changes. Don't over-engineer.
- After finishing, summarize what changed."""
```

복잡한 로직 없이 명확한 지시만 있습니다.

## 왜 이 설계가 작동하는가

**1. 단순함**
상태 머신 없음. 계획 모듈 없음. 프레임워크 없음.

**2. 모델이 생각**
모델이 어떤 도구를, 어떤 순서로, 언제 멈출지 결정합니다.

**3. 투명성**
모든 도구 호출이 보임. 모든 결과가 대화에 있음.

**4. 확장성**
도구 추가 = 함수 하나 + JSON 스키마 하나.

## 빠진 것들

| 기능 | 생략 이유 | 추가되는 버전 |
|------|----------|--------------|
| 할일 추적 | 필수 아님 | v2 |
| 서브에이전트 | 복잡도 | v3 |
| 권한 | 학습용으로 모델 신뢰 | 프로덕션 |

요점: **핵심은 작습니다**. 나머지는 모두 개선입니다.

## 더 큰 그림

Claude Code, Cursor Agent, Codex CLI, Devin—모두 이 패턴을 공유합니다:

```python
while not done:
    response = model(conversation, tools)
    results = execute(response.tool_calls)
    conversation.append(results)
```

차이점은 도구, 표시, 안전성에 있습니다. 하지만 본질은 항상: **모델에게 도구를 주고 일하게 하라**.

---

**모델이 곧 에이전트. 이것이 전체 비밀입니다.**

[← v0](./v0-bash-is-all-you-need_kr.md) | [README로 돌아가기](../README.md) | [v2 →](./v2-structured-planning_kr.md)

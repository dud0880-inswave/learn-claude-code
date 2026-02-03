# v0: Bash만 있으면 됩니다

**궁극의 단순화: ~50줄, 도구 1개, 완전한 에이전트 기능.**

v1, v2, v3을 구축한 후, 질문이 떠오릅니다: 에이전트의 *본질*은 무엇인가?

v0은 거꾸로 가면서 이 질문에 답합니다—핵심만 남을 때까지 모든 것을 벗겨냅니다.

## 핵심 인사이트

유닉스 철학: 모든 것은 파일이고, 모든 것은 파이프로 연결할 수 있습니다. Bash는 이 세계로 가는 관문입니다:

| 필요한 것 | Bash 명령어 |
|----------|------------|
| 파일 읽기 | `cat`, `head`, `grep` |
| 파일 쓰기 | `echo '...' > file` |
| 검색 | `find`, `grep`, `rg` |
| 실행 | `python`, `npm`, `make` |
| **서브에이전트** | `python v0_bash_agent.py "task"` |

마지막 줄이 핵심 인사이트입니다: **bash를 통해 자기 자신을 호출하면 서브에이전트가 구현됩니다**. Task 도구도, Agent Registry도 필요 없습니다—그냥 재귀입니다.

## 전체 코드

```python
#!/usr/bin/env python
from anthropic import Anthropic
import subprocess, sys, os

client = Anthropic(api_key="your-key", base_url="...")
TOOL = [{
    "name": "bash",
    "description": """Execute shell command. Patterns:
- Read: cat/grep/find/ls
- Write: echo '...' > file
- Subagent: python v0_bash_agent.py 'task description'""",
    "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}
}]
SYSTEM = f"CLI agent at {os.getcwd()}. Use bash. Spawn subagent for complex tasks."

def chat(prompt, history=[]):
    history.append({"role": "user", "content": prompt})
    while True:
        r = client.messages.create(model="...", system=SYSTEM, messages=history, tools=TOOL, max_tokens=8000)
        history.append({"role": "assistant", "content": r.content})
        if r.stop_reason != "tool_use":
            return "".join(b.text for b in r.content if hasattr(b, "text"))
        results = []
        for b in r.content:
            if b.type == "tool_use":
                out = subprocess.run(b.input["command"], shell=True, capture_output=True, text=True, timeout=300)
                results.append({"type": "tool_result", "tool_use_id": b.id, "content": out.stdout + out.stderr})
        history.append({"role": "user", "content": results})

if __name__ == "__main__":
    if len(sys.argv) > 1:
        print(chat(sys.argv[1]))  # Subagent mode
    else:
        h = []
        while (q := input(">> ")) not in ("q", ""):
            print(chat(q, h))
```

이것이 전체 에이전트입니다. ~50줄.

## 서브에이전트 작동 방식

```
Main Agent
  └─ bash: python v0_bash_agent.py "analyze architecture"
       └─ Subagent (격리된 프로세스, 새로운 히스토리)
            ├─ bash: find . -name "*.py"
            ├─ bash: cat src/main.py
            └─ stdout을 통해 요약 반환
```

**프로세스 격리 = 컨텍스트 격리**
- 자식 프로세스는 자체 `history=[]`를 가짐
- 부모는 stdout을 도구 결과로 캡처
- 재귀 호출로 무제한 중첩 가능

## v0이 포기한 것

| 기능 | v0 | v3 |
|------|----|----|
| 에이전트 타입 | 없음 | explore/code/plan |
| 도구 필터링 | 없음 | 화이트리스트 |
| 진행 상황 표시 | 일반 stdout | 인라인 업데이트 |
| 코드 복잡도 | ~50줄 | ~450줄 |

## v0이 증명하는 것

**복잡한 기능은 단순한 규칙에서 나온다:**

1. **도구 하나면 충분** — Bash는 모든 것으로 가는 관문
2. **재귀 = 계층구조** — 자기 호출이 서브에이전트를 구현
3. **프로세스 = 격리** — OS가 컨텍스트 분리를 제공
4. **프롬프트 = 제약** — 지시가 행동을 형성

핵심 패턴은 절대 변하지 않습니다:

```python
while True:
    response = model(messages, tools)
    if response.stop_reason != "tool_use":
        return response.text
    results = execute(response.tool_calls)
    messages.append(results)
```

나머지 모든 것—할일, 서브에이전트, 권한—은 이 루프를 둘러싼 개선입니다.

---

**Bash만 있으면 됩니다.**

[← README로 돌아가기](../README.md) | [v1 →](./v1-model-as-agent_kr.md)

# v2: Todo를 활용한 구조화된 계획

**~300줄. +1 도구. 명시적 작업 추적.**

v1은 작동합니다. 하지만 복잡한 작업에서 모델은 진행 상황을 잃어버릴 수 있습니다.

"인증 리팩토링, 테스트 추가, 문서 업데이트"를 요청하고 무슨 일이 일어나는지 보세요. 명시적 계획 없이는 작업 사이를 오가고, 단계를 잊고, 집중력을 잃습니다.

v2는 한 가지를 추가합니다: **Todo 도구**. 에이전트 작동 방식을 근본적으로 바꾸는 ~100줄의 새 코드입니다.

## 문제점

v1에서 계획은 모델의 "머릿속"에만 존재합니다:

```
v1: "A를 하고, 그 다음 B, 그 다음 C"  (보이지 않음)
    10개 도구 후: "잠깐, 내가 뭘 하고 있었지?"
```

Todo 도구는 이것을 명시적으로 만듭니다:

```
v2:
  [ ] 인증 모듈 리팩토링
  [>] 유닛 테스트 추가         <- 현재 여기
  [ ] 문서 업데이트
```

이제 당신과 모델 모두 계획을 볼 수 있습니다.

## TodoManager

제약이 있는 리스트:

```python
class TodoManager:
    def __init__(self):
        self.items = []  # 최대 20개

    def update(self, items):
        # 유효성 검사:
        # - 각각 필요: content, status, activeForm
        # - Status: pending | in_progress | completed
        # - in_progress는 하나만 가능
        # - 중복 없음, 빈 항목 없음
```

제약이 중요합니다:

| 규칙 | 이유 |
|------|------|
| 최대 20개 항목 | 무한 리스트 방지 |
| in_progress 하나만 | 집중 강제 |
| 필수 필드 | 구조화된 출력 |

이것들은 임의적이지 않습니다—가드레일입니다.

## 도구

```python
{
    "name": "TodoWrite",
    "input_schema": {
        "items": [{
            "content": "작업 설명",
            "status": "pending | in_progress | completed",
            "activeForm": "현재 시제: '파일 읽는 중'"
        }]
    }
}
```

`activeForm`은 지금 무엇이 일어나고 있는지 보여줍니다:

```
[>] 인증 코드 읽는 중...  <- activeForm
[ ] 유닛 테스트 추가
```

## 시스템 리마인더

todo 사용을 권장하는 부드러운 제약:

```python
INITIAL_REMINDER = "<reminder>다단계 작업에는 TodoWrite를 사용하세요.</reminder>"
NAG_REMINDER = "<reminder>todo 없이 10턴 이상. 업데이트해 주세요.</reminder>"
```

명령이 아닌 컨텍스트로 주입:

```python
# INITIAL_REMINDER: 대화 시작 시 (main에서)
if first_message:
    inject_reminder(INITIAL_REMINDER)

# NAG_REMINDER: agent_loop 내부에서, 작업 실행 중
if rounds_without_todo > 10:
    inject_reminder(NAG_REMINDER)
```

핵심 인사이트: NAG_REMINDER는 **에이전트 루프 내부에서** 주입되므로 모델은
장기 실행 작업 중에 이를 보게 됩니다, 작업 사이가 아니라.

## 피드백 루프

모델이 `TodoWrite`를 호출하면:

```
입력:
  [x] 인증 리팩토링 (completed)
  [>] 테스트 추가 (in_progress)
  [ ] 문서 업데이트 (pending)

반환:
  "[x] 인증 리팩토링
   [>] 테스트 추가
   [ ] 문서 업데이트
   (1/3 완료)"
```

모델은 자신의 계획을 봅니다. 업데이트합니다. 컨텍스트와 함께 계속합니다.

## Todo가 도움되는 경우

모든 작업에 필요한 것은 아닙니다:

| 적합한 경우 | 이유 |
|------------|------|
| 다단계 작업 | 추적할 5개 이상의 단계 |
| 긴 대화 | 20개 이상의 도구 호출 |
| 복잡한 리팩토링 | 여러 파일 |
| 교육 | 보이는 "생각" |

경험칙: **체크리스트를 작성할 것 같으면 todo를 사용하세요**.

## 통합

v2는 v1을 변경하지 않고 추가합니다:

```python
# v1 도구
tools = [bash, read_file, write_file, edit_file]

# v2 추가
tools.append(TodoWrite)
todo_manager = TodoManager()

# v2 사용량 추적
if rounds_without_todo > 10:
    inject_reminder()
```

~100줄의 새 코드. 같은 에이전트 루프.

## 더 깊은 인사이트

> **구조는 제약하고 가능하게 합니다.**

Todo 제약 (최대 항목, in_progress 하나)은 (보이는 계획, 추적되는 진행)을 가능하게 합니다.

에이전트 설계의 패턴:
- `max_tokens` 제약 → 관리 가능한 응답 가능
- 도구 스키마 제약 → 구조화된 호출 가능
- Todo 제약 → 복잡한 작업 완료 가능

좋은 제약은 한계가 아닙니다. 비계입니다.

---

**명시적 계획이 에이전트를 신뢰할 수 있게 만듭니다.**

[← v1](./v1-model-as-agent_kr.md) | [README로 돌아가기](../README.md) | [v3 →](./v3-subagent-mechanism_kr.md)

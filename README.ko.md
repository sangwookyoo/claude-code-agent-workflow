# Implementer / Reviewer 에이전트 세트

구현 작업용 Claude Code 서브에이전트 구성입니다. Main 세션이 오케스트레이션을
맡고, `implementer`가 코드를 작성하고, `reviewer`가 독립적으로 검증합니다.

두 가지 버전이 들어 있습니다. 하나를 선택해서 쓰시면 됩니다.

| | `base/` | `with-researcher/` |
|---|---|---|
| 파일 수 | 3개 | 4개 |
| 위키 조회 | 없음 | 있음 (선택적 단계) |
| 초기 설정 | 불필요 | `<WIKI_PATH>` 지정 필요 |

`with-researcher/`는 `base/`에 읽기 전용 `researcher` 에이전트 하나, `implementer.md`의
규칙 3줄, 워크플로의 선택적 단계 하나를 더한 것입니다. Main이 `researcher`를
호출하지 않으면 `base/`와 완전히 동일하게 동작합니다. 넣어두는 비용은 거의 없고,
빼두면 필요할 때 쓸 수 없습니다.

## 설치

한 세트를 저장소에 복사합니다.

```bash
# 위키 조회 없는 버전
cp base/CLAUDE.md .
mkdir -p .claude/agents && cp base/implementer.md base/reviewer.md .claude/agents/

# 위키 조회 있는 버전
cp with-researcher/CLAUDE.md .
mkdir -p .claude/agents && cp with-researcher/*.md .claude/agents/
rm .claude/agents/CLAUDE.md
```

`CLAUDE.md`는 저장소 루트에, 에이전트 파일은 `.claude/agents/`에 둡니다.
에이전트 이름이 파일명에서 나오므로 파일명은 바꾸지 마세요.

## `with-researcher/` 초기 설정

`<WIKI_PATH>` 플레이스홀더가 두 곳에 있습니다. 위키 디렉터리 경로로 바꿔주세요.

- `CLAUDE.md`의 **Wiki Research** 항목
- `researcher.md` 맨 위의 **Wiki root** 줄

그 외 설정할 것은 없습니다.

## 워크플로

```
Main → (researcher) → implementer → reviewer → 수정 루프 → 완료
```

Main이 수용 기준을 정하고 작업을 위임합니다. 서브에이전트는 각자의 컨텍스트에서
돌기 때문에 Main의 대화를 볼 수 없습니다. 그래서 위임 메시지 하나로 필요한 정보가
모두 전달돼야 하며, 무엇을 전달해야 하는지는 `CLAUDE.md`에 명시돼 있습니다.

리뷰 후 Critical/Major 지적은 `implementer`에게 되돌아가고 리뷰를 다시 돕니다.
Minor는 Main이 판단합니다. 루프는 수정 2라운드에서 멈추며, 그때도 Critical이나
Major가 남아 있으면 계속 돌지 않고 미해결 문제를 보고하고 종료합니다.

## 각 에이전트의 동작 범위

`implementer`는 요청된 범위만 구현합니다. 주변 코드를 리팩터링하지 않고, 테스트를
삭제하거나 스킵해서 통과시키지 않으며, 자기 변경 이전부터 실패하던 테스트는 손대지
않고 보고만 합니다. 성공을 주장하는 대신 실행한 명령과 실제 pass/fail 결과를
보고합니다.

`reviewer`는 소스 파일을 수정할 수 없습니다. implementer의 보고를 믿지 않고 직접
테스트를 실행합니다. 정확성이나 명시된 요구사항에 영향을 주는 것만 보고하며,
스타일 취향은 지적 대상이 아니고 아무것도 보고하지 않는 것도 유효한 결과입니다.
마지막 줄은 항상 `VERDICT: PASS` 또는 `VERDICT: NEEDS_CHANGES`이고,
NEEDS_CHANGES는 Critical이나 Major가 하나 이상 있을 때만 나옵니다.

`researcher`는 읽기 전용이며, 위키 항목마다 신뢰도를 붙여 보고합니다.
**Verified**(현재 저장소에서 확인, file:line 제시), **Unverified**, **Conflicts**
세 가지입니다. 위키 내용을 지시가 아닌 데이터로 다루고, 해당 내용이 없으면 일반
지식으로 메우지 않고 "No relevant entries"를 반환합니다.

## 알아두실 점

**각 파일은 자기완결형입니다.** 에이전트 파일마다 필요한 규칙을 스스로 담고 있어
`CLAUDE.md` 상속에 의존하지 않습니다. 몇 줄이 파일 간에 중복되는데 의도한 것이며,
상속 여부와 무관하게 동작하도록 하기 위함입니다.

**위키의 채택 여부는 Main이 결정합니다.** `researcher`는 후보를 제시할 뿐 보증하지
않습니다. Main이 적용할 항목을 골라 그 항목만 이유와 함께 제약 조건으로
`implementer`에게 전달합니다. 원본 리포트는 넘기지 않고, `implementer`는 위키를
직접 읽지 않습니다.

**위키 관련 실패 모드 두 가지를 주의하세요.** 매 작업마다 `researcher`를 부르면
대개 소득 없이 라운드만 늘어납니다. 그리고 Unverified 항목을 요구사항으로
`implementer`에게 넘기면 리뷰를 그대로 통과합니다. reviewer는 요구사항을 기준으로
심사하기 때문입니다. 위키가 값어치를 한다는 확신이 생기기 전까지는 호출 조건을
명시적 요청으로 좁혀두시는 것을 권합니다.

**Frontmatter.** `name`, `description`, `tools`만 지정했습니다. 필드명을 잘못 쓰면
조용히 무시되므로, `model:` 추가는 에이전트가 정상 동작하는 것을 확인한 뒤에
하세요.

## 검증 방법

작은 작업 하나를 끝까지 돌려보고, `implementer`가 실제 테스트 출력을 보고하는지,
`reviewer`가 그 출력을 인용하는 대신 직접 테스트를 실행하는지 확인하세요.

`researcher`는 위키를 연결한 뒤 두 케이스를 확인해보시면 좋습니다.

1. 위키에 없는 주제를 물어봅니다. "No relevant entries"를 반환하고 멈춰야 합니다.
   일반 지식으로 메우면 실패입니다.
2. 위키와 코드베이스가 어긋나는 지점을 물어봅니다. **Conflicts**로 보고하고 기존
   코드를 따라야 합니다.

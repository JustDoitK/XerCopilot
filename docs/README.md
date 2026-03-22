# GitHub Copilot 활용 가이드

> VSCode에서 GitHub Copilot을 효과적으로 사용하기 위한 팀 가이드

---

## 문서 목록

| 문서 | 내용 |
|------|------|
| [01. 개요](./01-overview.md) | Copilot 기능 전체 구성, 단축키, 설치 방법 |
| [02. 코드 자동완성](./02-code-completion.md) | Inline Suggestion 활용 패턴, 컨텍스트 전략 |
| [03. Copilot Chat](./03-copilot-chat.md) | Chat, Inline Chat, @에이전트, 슬래시 명령어 |
| [04. 프롬프트 엔지니어링](./04-prompt-engineering.md) | 좋은 프롬프트 작성법, 상황별 템플릿 |
| [05. 커스텀 지시사항](./05-custom-instructions.md) | copilot-instructions.md, 팀 규칙 자동 적용 |
| [06. 커스텀 에이전트 개발](./06-custom-agent-extension.md) | 회사 전용 @에이전트 만들기 (Jira, DB, 배포 등) |
| [07. @confluence 에이전트](./07-confluence-agent.md) | Confluence 문서 검색/생성/수정 에이전트 개발 |

---

## 빠른 시작

### 오늘 바로 써볼 수 있는 것들

**1. 주석으로 코드 생성**
```python
# 이메일 유효성 검사 함수
def validate_email(email):
    # Tab 키로 자동 완성
```

**2. 코드 선택 후 Inline Chat**
- 코드 블록 선택 → `Ctrl + I` → "이 코드 설명해줘" 또는 "/fix"

**3. Copilot Chat에서 프로젝트 질문**
```
@workspace 이 프로젝트 구조 설명해줘
@workspace 결제 관련 코드가 어디 있어?
```

**4. 슬래시 명령어**
```
/explain  → 코드 설명
/fix      → 버그 수정
/test     → 테스트 생성
/doc      → 문서 주석 생성
```

---

## 핵심 단축키 요약

| 단축키 | 기능 |
|--------|------|
| `Tab` | 자동완성 제안 수락 |
| `Ctrl + I` | Inline Chat 열기 |
| `Ctrl + Shift + I` | Copilot Chat 패널 열기 |
| `Ctrl + Enter` | 여러 제안 목록 보기 |
| `Alt + ]` / `Alt + [` | 다음/이전 제안 |

---

## 팀 적용 우선순위

1. **즉시** → `.github/copilot-instructions.md` 파일에 팀 코딩 규칙 작성
2. **이번 주** → 슬래시 명령어와 `@workspace` 에이전트 익히기
3. **이번 달** → 프롬프트 엔지니어링 패턴 익히고 `.prompt.md` 파일 만들기

---

## 더 알아보기

- [GitHub Copilot 공식 문서](https://docs.github.com/en/copilot)
- [VSCode Copilot 확장 마켓플레이스](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot)

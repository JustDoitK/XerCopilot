# GitHub Copilot 개요

## GitHub Copilot이란?

GitHub Copilot은 OpenAI의 Codex 모델을 기반으로 한 AI 코드 보조 도구입니다.
VSCode에서 사용할 경우 코드 자동완성, 채팅, 코드 리뷰 등 다양한 기능을 제공합니다.

## 주요 기능 구성

| 기능 | 설명 |
|------|------|
| **Inline Suggestion** | 코드 작성 중 실시간으로 다음 코드를 제안 |
| **Copilot Chat** | 채팅 형식으로 코드 관련 질문 및 요청 처리 |
| **Inline Chat** | 에디터 내에서 선택 영역에 대해 직접 질문/수정 요청 |
| **@참조 (Agents)** | `@workspace`, `@terminal`, `@vscode` 등 범위별 전문 에이전트 |
| **Slash Commands** | `/fix`, `/explain`, `/test`, `/doc` 등 단축 명령어 |
| **Custom Instructions** | 프로젝트/팀 맞춤 지시사항 설정 |
| **Copilot Edits** | 여러 파일에 걸친 변경을 한 번에 요청 |

## VSCode 확장 설치

1. VSCode 마켓플레이스에서 **GitHub Copilot** 검색 후 설치
2. **GitHub Copilot Chat**도 함께 설치
3. GitHub 계정으로 로그인 (Copilot 구독 필요)

## 빠른 시작 단축키

| 단축키 | 기능 |
|--------|------|
| `Tab` | 제안 수락 |
| `Esc` | 제안 거절 |
| `Alt + ]` / `Alt + [` | 다음/이전 제안 보기 |
| `Ctrl + Enter` | 여러 제안 목록 열기 |
| `Ctrl + I` | Inline Chat 열기 |
| `Ctrl + Shift + I` | Copilot Chat 패널 열기 |
| `Ctrl + Shift + Alt + L` | Copilot Edits 열기 |

## 파일 구성 권장 사항

```
프로젝트 루트/
├── .github/
│   └── copilot-instructions.md   ← 프로젝트 전체 커스텀 지시사항
├── docs/
│   ├── 01-overview.md
│   ├── 02-code-completion.md
│   ├── 03-copilot-chat.md
│   ├── 04-prompt-engineering.md
│   └── 05-custom-instructions.md
└── ...
```

> **Tip**: `.github/copilot-instructions.md` 파일을 만들면 Copilot이 항상 그 내용을 참고합니다.

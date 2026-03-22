# Copilot Chat & Inline Chat 활용

## Chat vs Inline Chat

| 구분 | Copilot Chat (`Ctrl+Shift+I`) | Inline Chat (`Ctrl+I`) |
|------|-------------------------------|------------------------|
| 위치 | 사이드 패널 | 에디터 내 팝업 |
| 용도 | 질문, 설명 요청, 새 코드 생성 | 선택한 코드 수정/설명 |
| 맥락 | `@`, `#` 참조로 범위 지정 | 선택 영역 자동 참조 |

---

## @ 에이전트 (Agents)

채팅 입력창에서 `@`로 에이전트를 지정해 범위를 좁힐 수 있습니다.

### @workspace
프로젝트 전체 코드베이스를 검색해서 답변합니다.

```
@workspace 이 프로젝트에서 인증 로직이 어디에 있어?
@workspace UserService 클래스가 어디서 사용되고 있어?
@workspace 데이터베이스 연결 설정은 어떻게 되어 있어?
```

### @terminal
현재 터미널 상태와 연동해 명령어를 제안합니다.

```
@terminal 이 에러를 해결하려면 어떤 명령어를 실행해야 해?
@terminal 의존성 설치 명령어 알려줘
```

### @vscode
VSCode 설정 및 기능에 대해 답변합니다.

```
@vscode 디버거 설정 방법 알려줘
@vscode 특정 확장을 설치하는 방법은?
```

---

## # 파일 참조

`#`으로 특정 파일이나 심볼을 채팅 컨텍스트에 포함시킵니다.

```
#file:userService.ts 이 파일에서 에러 처리가 미흡한 부분 찾아줘

#UserService 이 클래스에 캐싱 로직 추가하면 어떨까?

#selection 선택한 이 코드를 리팩토링해줘
```

---

## 슬래시 명령어 (Slash Commands)

채팅창에서 `/`로 시작하는 명령어로 빠르게 작업을 지시합니다.

### /explain — 코드 설명

```
/explain 이 함수가 어떻게 동작하는지 설명해줘

// 코드 선택 후
/explain
```

### /fix — 버그 수정

```
/fix
// → 선택된 코드나 현재 에러를 분석해서 수정 제안

// 에러 메시지와 함께
/fix TypeError: Cannot read property 'id' of undefined
```

### /test — 테스트 코드 생성

```
/test
// → 현재 함수/클래스에 대한 단위 테스트 자동 생성

/test Jest를 사용해서 테스트 코드 작성해줘
```

### /doc — 문서 주석 생성

```
/doc
// → JSDoc, docstring 등 언어에 맞는 문서 주석 자동 생성
```

### /new — 새 파일/프로젝트 생성

```
/new Express.js REST API 보일러플레이트 만들어줘
/new React 컴포넌트 파일 생성해줘
```

### /optimize — 성능 최적화

```
/optimize 이 쿼리 성능을 개선할 방법 제안해줘
```

---

## Copilot Edits (멀티 파일 편집)

`Ctrl+Shift+Alt+L`로 Copilot Edits를 열면 여러 파일에 걸친 변경을 한 번에 요청할 수 있습니다.

```
// 예시 요청
"users 테이블에 phone_number 컬럼을 추가하고,
 관련된 모델, 서비스, API, 테스트 파일 모두 업데이트해줘"

// → Copilot이 관련 파일들을 찾아 일괄 수정 제안
```

변경 사항은 **diff로 미리보기** 후 수락/거절할 수 있습니다.

---

## 실전 활용 예시

### 코드 리뷰 요청
```
#file:pullRequest.ts /explain 이 코드의 잠재적인 문제점을 찾아줘
```

### 레거시 코드 이해
```
@workspace 이 프로젝트에서 결제 처리 플로우를 전체적으로 설명해줘
```

### 리팩토링 제안
```
#selection 이 코드를 SOLID 원칙에 맞게 리팩토링해줘
```

### SQL 쿼리 최적화
```
아래 쿼리를 최적화해줘:
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at > '2024-01-01'
```

### 에러 디버깅
```
/fix
Uncaught TypeError: Cannot read properties of null (reading 'map')
at UserList (UserList.tsx:23)
```

# 코드 자동완성 (Inline Suggestion) 활용

## 기본 동작 방식

Copilot은 현재 파일의 내용(코드, 주석, 파일명)을 컨텍스트로 읽어 다음에 올 코드를 제안합니다.
**회색 글자**로 표시되는 제안을 `Tab`으로 수락하거나 `Esc`로 거절합니다.

---

## 효과적인 활용 패턴

### 1. 주석으로 의도 설명 후 코드 생성

주석을 먼저 작성하면 Copilot이 그 의도에 맞는 코드를 생성합니다.

```python
# 사용자 목록에서 활성 상태인 사용자만 필터링하여 이메일 리스트 반환
def get_active_user_emails(users):
    # Copilot이 자동으로 아래 코드를 제안
    return [user.email for user in users if user.is_active]
```

### 2. 함수 시그니처 작성 후 구현 생성

함수 이름과 파라미터를 명확하게 작성하면 본문을 자동 완성합니다.

```typescript
// 명확한 이름 → 좋은 제안
async function fetchUserById(userId: string): Promise<User> {
    // Copilot이 fetch 로직을 자동 생성
}

// 모호한 이름 → 나쁜 제안
async function getData(id) {
    // 어떤 데이터인지 알 수 없어 제안 품질 저하
}
```

### 3. 테스트 코드 자동 생성

구현 코드 옆에 테스트 파일을 열어두면 Copilot이 맥락을 파악합니다.

```python
# test_calculator.py
def test_add_two_positive_numbers():
    # Copilot이 assert 구문을 자동 제안
    assert add(2, 3) == 5

def test_add_negative_number():
    # 패턴을 학습해 다음 테스트도 자동 완성
```

### 4. 반복 패턴에서 자동 완성

유사한 코드가 반복될 때 Copilot이 패턴을 학습해 나머지를 완성합니다.

```javascript
const routes = [
    { path: '/home', component: HomePage, exact: true },
    { path: '/about', component: AboutPage, exact: true },
    // 여기서 Tab → 다음 라우트 자동 완성
];
```

### 5. 여러 줄 제안 보기

`Ctrl + Enter`를 누르면 여러 가지 대안 제안을 한눈에 볼 수 있습니다.

---

## 컨텍스트를 잘 주는 방법

Copilot은 다음 요소들을 컨텍스트로 활용합니다:

- **현재 파일 내용** (가장 중요)
- **열려 있는 탭의 파일들** - 관련 파일을 함께 열어두면 더 정확한 제안
- **파일 이름과 경로** - `userService.ts`라는 이름만으로도 유저 관련 코드 생성 유도
- **임포트 구문** - 어떤 라이브러리를 쓰는지 파악

### 관련 파일 열어두기 전략

```
작업 중인 파일: userController.ts
함께 열어두기:  userService.ts, userRepository.ts, user.model.ts
→ Copilot이 프로젝트의 패턴과 타입을 파악해 더 정확한 코드 생성
```

---

## 자동완성 설정 조정

`settings.json`에서 동작을 제어할 수 있습니다:

```json
{
    // 특정 언어에서만 활성화
    "github.copilot.enable": {
        "*": true,
        "plaintext": false,
        "markdown": true
    },
    // 제안 표시 딜레이 (ms)
    "editor.inlineSuggest.enabled": true
}
```

---

## 단축키 요약

| 단축키 | 동작 |
|--------|------|
| `Tab` | 제안 전체 수락 |
| `Ctrl + →` | 단어 단위로 부분 수락 |
| `Esc` | 제안 거절 |
| `Alt + ]` | 다음 제안 |
| `Alt + [` | 이전 제안 |
| `Ctrl + Enter` | 여러 제안 목록 열기 |

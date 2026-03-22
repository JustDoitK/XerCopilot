# 커스텀 지시사항 (Custom Instructions) 설정

## 개요

GitHub Copilot에 **팀/프로젝트 공통 규칙을 사전 설정**해두면,
매번 같은 지시를 반복하지 않아도 됩니다.

---

## 방법 1: `.github/copilot-instructions.md` (권장)

프로젝트 루트에 이 파일을 만들면 **Copilot Chat이 항상 이 내용을 참고**합니다.
팀 전체가 Git으로 공유할 수 있어 가장 효과적입니다.

### 파일 위치
```
프로젝트 루트/
└── .github/
    └── copilot-instructions.md   ← 이 파일 생성
```

### 작성 예시

```markdown
# 프로젝트 Copilot 지시사항

## 기술 스택
- Backend: Node.js 18 + TypeScript 5 + Express 4
- Database: PostgreSQL 15 + TypeORM
- Testing: Jest + Supertest
- 코드 스타일: ESLint (Airbnb) + Prettier

## 코딩 규칙
- 모든 함수는 TypeScript 타입을 명시할 것
- async 함수는 반드시 try/catch로 에러 처리
- 로깅은 `logger` 모듈 사용 (console.log 금지)
- 환경변수는 `config/env.ts`를 통해서만 접근

## 네이밍 규칙
- 변수/함수: camelCase
- 클래스/인터페이스: PascalCase
- 상수: UPPER_SNAKE_CASE
- 파일: kebab-case

## 폴더 구조
- 컨트롤러: src/controllers/
- 서비스: src/services/
- 레포지토리: src/repositories/
- 모델/엔티티: src/entities/
- 미들웨어: src/middlewares/

## API 응답 형식
성공: { success: true, data: {...} }
에러: { success: false, error: { code: string, message: string } }

## 테스트 규칙
- 파일명: *.spec.ts
- describe 블록으로 기능별 그룹화
- 각 테스트는 독립적으로 실행 가능해야 함
```

---

## 방법 2: VSCode 설정 (사용자 전체 적용)

`settings.json`에 개인 지시사항을 설정합니다.
모든 프로젝트에 공통으로 적용됩니다.

```json
{
    "github.copilot.chat.codeGeneration.instructions": [
        {
            "text": "항상 TypeScript를 사용하고, any 타입 사용을 피해줘"
        },
        {
            "text": "에러 처리는 항상 포함해줘"
        },
        {
            "text": "코드 설명은 한국어로 해줘"
        }
    ],
    "github.copilot.chat.testGeneration.instructions": [
        {
            "text": "Jest 프레임워크를 사용해서 테스트를 작성해줘"
        },
        {
            "text": "given-when-then 패턴으로 테스트를 구성해줘"
        }
    ]
}
```

---

## 방법 3: .instructions.md 파일 (폴더별 적용)

특정 폴더에 `.instructions.md` 파일을 두면 해당 폴더의 파일 작업 시 자동 적용됩니다.

```
src/
├── api/
│   └── .instructions.md   ← API 관련 규칙
├── utils/
│   └── .instructions.md   ← 유틸 함수 관련 규칙
└── tests/
    └── .instructions.md   ← 테스트 관련 규칙
```

**`src/api/.instructions.md` 예시:**
```markdown
---
applyTo: "**"
---

이 폴더의 모든 파일은 REST API 엔드포인트입니다.

규칙:
- 입력 유효성 검증 항상 포함
- HTTP 상태 코드 명확히 사용
- Swagger 주석 포함 (@swagger)
- Rate limiting 고려
```

---

## 방법 4: .prompt.md 파일 (재사용 프롬프트)

자주 쓰는 복잡한 프롬프트를 파일로 저장해두고 재사용합니다.

```
.github/
└── prompts/
    ├── code-review.prompt.md
    ├── write-tests.prompt.md
    └── api-endpoint.prompt.md
```

**`code-review.prompt.md` 예시:**
```markdown
---
mode: 'ask'
---

아래 코드를 시니어 개발자 관점에서 리뷰해줘.

체크리스트:
- [ ] 보안 취약점 (SQL Injection, XSS, 인증/인가 누락)
- [ ] 성능 문제 (N+1 쿼리, 불필요한 루프)
- [ ] 에러 처리 누락
- [ ] 코드 중복
- [ ] 테스트 가능성
- [ ] 네이밍 적절성

각 항목에 대해 심각도(높음/중간/낮음)와 함께 설명해줘.
```

채팅에서 `#file:.github/prompts/code-review.prompt.md`로 불러오거나,
VSCode Command Palette에서 "Copilot: Run Prompt"로 실행합니다.

---

## 커스텀 지시사항 우선순위

```
높음  ← 채팅 입력창에서 직접 작성한 지시사항
      ← .instructions.md (폴더별)
      ← .github/copilot-instructions.md (프로젝트)
낮음  ← settings.json (사용자 전체)
```

충돌 시 더 구체적인(높은 우선순위) 지시사항이 적용됩니다.

---

## 팀에서 활용하는 전략

### 온보딩 자동화
```markdown
# .github/copilot-instructions.md

## 신규 팀원을 위한 안내
이 프로젝트는 도메인 주도 설계(DDD)를 따릅니다.
- 비즈니스 로직은 반드시 서비스 레이어에만 작성
- 컨트롤러는 요청/응답 변환만 담당
- 레포지토리는 DB 접근만 담당
```

### 일관된 코드 품질 유지
팀의 코딩 컨벤션, 아키텍처 패턴, 보안 요구사항을 `.github/copilot-instructions.md`에 문서화하면
Copilot이 자동으로 팀 스타일에 맞는 코드를 생성합니다.

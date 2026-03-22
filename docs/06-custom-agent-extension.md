# GitHub Copilot Extension으로 커스텀 에이전트 만들기

## 개요

**GitHub Copilot Extension**을 사용하면 Copilot Chat에서 호출 가능한
회사 전용 `@에이전트`를 직접 만들 수 있습니다.

```
// 예시: 커스텀 에이전트 호출
@jira 이번 스프린트 남은 티켓 목록 보여줘
@confluence 배포 프로세스 문서 찾아줘
@db users 테이블 스키마 알려줘
@deploy 현재 prod 배포 상태는?
```

---

## 동작 방식

```
[VSCode Copilot Chat]
        │
        │ @my-agent "질문 내용"
        ▼
[GitHub App (Copilot Extension)]
        │
        │ HTTP POST (메시지 전달)
        ▼
[우리가 만든 서버]
        │
        │ 내부 API 호출, DB 조회, 문서 검색 등
        ▼
[스트리밍 응답 반환]
        │
        ▼
[Copilot Chat에 결과 표시]
```

---

## 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **GitHub App** | 에이전트를 등록하는 GitHub 앱 |
| **Webhook 서버** | 메시지를 받아서 처리하는 백엔드 |
| **SSE 스트리밍** | 응답을 실시간으로 스트리밍하는 방식 |

---

## Step 1: GitHub App 생성

1. GitHub → Settings → Developer settings → **GitHub Apps** → New GitHub App
2. 설정 항목:

```
앱 이름: my-company-copilot-agent
Homepage URL: https://your-server.com
Webhook URL: https://your-server.com/webhook

Permissions:
  - Copilot Chat: Read (필수)
  - 필요에 따라 추가 (예: Issues, Repositories)

Copilot:
  - [x] Copilot Extensions 활성화
```

---

## Step 2: 서버 구현 (Node.js + TypeScript)

### 프로젝트 구조

```
my-copilot-agent/
├── src/
│   ├── index.ts          # 서버 진입점
│   ├── handler.ts        # 메시지 처리 로직
│   └── tools/
│       ├── jira.ts       # Jira API 연동
│       ├── confluence.ts # Confluence 연동
│       └── database.ts   # DB 조회
├── package.json
└── tsconfig.json
```

### 기본 서버 코드

```typescript
// src/index.ts
import express from 'express';
import { handleCopilotMessage } from './handler';
import { verifyWebhookSignature } from './utils/auth';

const app = express();
app.use(express.json());

// Copilot Extension 엔드포인트
app.post('/agent', verifyWebhookSignature, async (req, res) => {
    // SSE 헤더 설정 (스트리밍 응답)
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');

    const { messages } = req.body;
    const userMessage = messages[messages.length - 1].content;

    await handleCopilotMessage(userMessage, res);
    res.end();
});

app.listen(3000, () => console.log('Copilot Agent running on port 3000'));
```

### 메시지 처리 및 SSE 스트리밍

```typescript
// src/handler.ts
import { Response } from 'express';

// SSE 형식으로 텍스트 전송
function sendSSE(res: Response, data: string) {
    res.write(`data: ${JSON.stringify({
        choices: [{
            delta: { content: data },
            finish_reason: null
        }]
    })}\n\n`);
}

// 스트리밍 종료
function endSSE(res: Response) {
    res.write(`data: ${JSON.stringify({
        choices: [{
            delta: {},
            finish_reason: 'stop'
        }]
    })}\n\n`);
    res.write('data: [DONE]\n\n');
}

export async function handleCopilotMessage(message: string, res: Response) {
    try {
        sendSSE(res, '분석 중...\n\n');

        // 키워드에 따라 다른 도구 호출
        if (message.includes('jira') || message.includes('티켓')) {
            const issues = await fetchJiraIssues(message);
            sendSSE(res, formatJiraResponse(issues));

        } else if (message.includes('배포') || message.includes('deploy')) {
            const status = await getDeployStatus();
            sendSSE(res, formatDeployStatus(status));

        } else if (message.includes('db') || message.includes('스키마')) {
            const schema = await getDatabaseSchema(message);
            sendSSE(res, formatSchemaResponse(schema));

        } else {
            sendSSE(res, '다음 명령을 사용할 수 있습니다:\n');
            sendSSE(res, '- `jira [검색어]`: Jira 티켓 조회\n');
            sendSSE(res, '- `배포 상태`: 현재 배포 상태 확인\n');
            sendSSE(res, '- `db [테이블명]`: DB 스키마 조회\n');
        }

        endSSE(res);

    } catch (error) {
        sendSSE(res, `오류가 발생했습니다: ${error.message}`);
        endSSE(res);
    }
}
```

### Jira 연동 예시

```typescript
// src/tools/jira.ts
import axios from 'axios';

const JIRA_BASE_URL = process.env.JIRA_BASE_URL;
const JIRA_TOKEN = process.env.JIRA_API_TOKEN;
const JIRA_EMAIL = process.env.JIRA_EMAIL;

export async function fetchJiraIssues(query: string): Promise<JiraIssue[]> {
    const jql = buildJQL(query); // 자연어를 JQL로 변환

    const response = await axios.get(`${JIRA_BASE_URL}/rest/api/3/search`, {
        headers: {
            Authorization: `Basic ${Buffer.from(`${JIRA_EMAIL}:${JIRA_TOKEN}`).toString('base64')}`,
            'Content-Type': 'application/json',
        },
        params: { jql, maxResults: 10 },
    });

    return response.data.issues;
}

function buildJQL(query: string): string {
    if (query.includes('내 티켓')) {
        return 'assignee = currentUser() AND status != Done ORDER BY priority DESC';
    }
    if (query.includes('이번 스프린트')) {
        return 'sprint in openSprints() ORDER BY priority DESC';
    }
    return `text ~ "${query}" ORDER BY created DESC`;
}
```

### 서명 검증 (보안)

```typescript
// src/utils/auth.ts
import crypto from 'crypto';
import { Request, Response, NextFunction } from 'express';

export function verifyWebhookSignature(req: Request, res: Response, next: NextFunction) {
    const signature = req.headers['x-github-token'] as string;
    const secret = process.env.WEBHOOK_SECRET;

    if (!secret) return next(); // 개발 환경

    const expectedSig = crypto
        .createHmac('sha256', secret)
        .update(JSON.stringify(req.body))
        .digest('hex');

    if (`sha256=${expectedSig}` !== signature) {
        return res.status(401).json({ error: 'Invalid signature' });
    }

    next();
}
```

---

## Step 3: 배포 및 등록

### 서버 배포 (예: Railway, Render, AWS)
```bash
# 환경변수 설정
JIRA_BASE_URL=https://yourcompany.atlassian.net
JIRA_API_TOKEN=your_token
JIRA_EMAIL=your@email.com
WEBHOOK_SECRET=your_webhook_secret
```

### GitHub App에 에이전트 URL 등록
```
GitHub App 설정 → Copilot → Agent URL:
https://your-deployed-server.com/agent
```

### 조직/팀에 설치
1. GitHub App → Install App → 회사 Organization 선택
2. 팀원들이 VSCode에서 GitHub 재로그인 후 사용 가능

---

## 여러 에이전트 운영 전략

에이전트를 기능별로 분리해서 각각 다른 역할을 맡길 수 있습니다.

```
@jira     → Jira/프로젝트 관리 도구 연동
@docs     → 사내 Confluence/Notion 문서 검색
@db       → 개발 DB 스키마/데이터 조회 (읽기 전용)
@deploy   → CI/CD 파이프라인 상태, 배포 이력
@review   → 내부 코드 스타일/보안 규칙 검사
```

### 구현 방식: 단일 서버 or 분리

```
// 방법 1: 단일 서버에서 라우팅
app.post('/jira',   handleJiraAgent);
app.post('/docs',   handleDocsAgent);
app.post('/deploy', handleDeployAgent);
// → GitHub App을 각각 따로 만들고 URL만 다르게 설정

// 방법 2: 각 에이전트를 독립 서비스로
jira-agent.company.com   → Jira 전용 서버
docs-agent.company.com   → 문서 검색 전용 서버
deploy-agent.company.com → 배포 전용 서버
```

---

## 로컬 개발 테스트

```bash
# ngrok으로 로컬 서버를 임시 외부 노출
npx ngrok http 3000

# GitHub App Webhook URL을 ngrok URL로 임시 설정
# https://abc123.ngrok.io/agent

# 서버 실행
npm run dev
```

---

## 참고 자료

- [GitHub Copilot Extensions 공식 문서](https://docs.github.com/en/copilot/building-copilot-extensions)
- [GitHub Copilot Extensions SDK](https://github.com/github/copilot-extensions-aat)
- [예제 Extension 모음](https://github.com/github/copilot-extensions)

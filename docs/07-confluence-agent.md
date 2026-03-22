# @confluence 커스텀 에이전트 개발 가이드

## 개요

Confluence REST API를 연동한 `@confluence` 에이전트를 만들면
VSCode Copilot Chat에서 직접 문서를 **검색, 읽기, 생성, 수정**할 수 있습니다.

```
// 사용 예시
@confluence "배포 프로세스" 문서 찾아줘
@confluence 오늘 작업한 내용으로 회의록 페이지 만들어줘
@confluence #file:api-spec.ts 이 파일 내용으로 API 문서 페이지 업데이트해줘
@confluence DEV 스페이스에 새 온보딩 가이드 초안 작성해줘
```

---

## Confluence API 사전 준비

### API 토큰 발급
1. `https://id.atlassian.com/manage-profile/security/api-tokens` 접속
2. **Create API token** → 토큰 이름 입력 → 복사

### 필요한 환경변수
```env
CONFLUENCE_BASE_URL=https://yourcompany.atlassian.net/wiki
CONFLUENCE_EMAIL=your@company.com
CONFLUENCE_API_TOKEN=your_api_token
```

### API 인증 방식 (Basic Auth)
```
Authorization: Basic base64(email:api_token)
```

---

## 프로젝트 구조

```
confluence-agent/
├── src/
│   ├── index.ts              # Express 서버
│   ├── handler.ts            # 메시지 파싱 및 라우팅
│   ├── confluence/
│   │   ├── client.ts         # Confluence API 클라이언트
│   │   ├── search.ts         # 페이지 검색
│   │   ├── reader.ts         # 페이지 읽기
│   │   ├── writer.ts         # 페이지 생성/수정
│   │   └── formatter.ts      # 마크다운 ↔ Confluence Storage Format 변환
│   └── utils/
│       ├── auth.ts           # 서명 검증
│       └── sse.ts            # SSE 헬퍼
├── package.json
└── tsconfig.json
```

---

## Step 1: Confluence API 클라이언트

```typescript
// src/confluence/client.ts
import axios, { AxiosInstance } from 'axios';

export class ConfluenceClient {
    private http: AxiosInstance;
    private baseUrl: string;

    constructor() {
        this.baseUrl = process.env.CONFLUENCE_BASE_URL!;
        const token = Buffer.from(
            `${process.env.CONFLUENCE_EMAIL}:${process.env.CONFLUENCE_API_TOKEN}`
        ).toString('base64');

        this.http = axios.create({
            baseURL: `${this.baseUrl}/rest/api`,
            headers: {
                Authorization: `Basic ${token}`,
                'Content-Type': 'application/json',
            },
        });
    }

    async searchPages(query: string, spaceKey?: string): Promise<ConfluencePage[]> {
        const cql = spaceKey
            ? `text ~ "${query}" AND space = "${spaceKey}"`
            : `text ~ "${query}"`;

        const res = await this.http.get('/content/search', {
            params: { cql, limit: 5, expand: 'body.storage,space,version' },
        });
        return res.data.results;
    }

    async getPage(pageId: string): Promise<ConfluencePage> {
        const res = await this.http.get(`/content/${pageId}`, {
            params: { expand: 'body.storage,version,space,ancestors' },
        });
        return res.data;
    }

    async createPage(params: CreatePageParams): Promise<ConfluencePage> {
        const res = await this.http.post('/content', {
            type: 'page',
            title: params.title,
            space: { key: params.spaceKey },
            ancestors: params.parentId ? [{ id: params.parentId }] : undefined,
            body: {
                storage: {
                    value: params.content, // Confluence Storage Format (XHTML)
                    representation: 'storage',
                },
            },
        });
        return res.data;
    }

    async updatePage(params: UpdatePageParams): Promise<ConfluencePage> {
        const current = await this.getPage(params.pageId);
        const res = await this.http.put(`/content/${params.pageId}`, {
            type: 'page',
            title: params.title ?? current.title,
            version: { number: current.version.number + 1 },
            body: {
                storage: {
                    value: params.content,
                    representation: 'storage',
                },
            },
        });
        return res.data;
    }

    async getSpaces(): Promise<ConfluenceSpace[]> {
        const res = await this.http.get('/space', { params: { limit: 20 } });
        return res.data.results;
    }
}

// 타입 정의
interface ConfluencePage {
    id: string;
    title: string;
    space: { key: string; name: string };
    version: { number: number };
    body: { storage: { value: string } };
    _links: { webui: string };
}

interface ConfluenceSpace {
    key: string;
    name: string;
}

interface CreatePageParams {
    title: string;
    spaceKey: string;
    content: string;     // Confluence Storage Format
    parentId?: string;
}

interface UpdatePageParams {
    pageId: string;
    content: string;
    title?: string;
}
```

---

## Step 2: 마크다운 ↔ Confluence 포맷 변환

Copilot이 생성하는 마크다운을 Confluence가 이해하는 Storage Format(XHTML)으로 변환합니다.

```typescript
// src/confluence/formatter.ts
import { marked } from 'marked';

// 마크다운 → Confluence Storage Format
export function markdownToConfluence(markdown: string): string {
    // marked로 HTML 변환 후 Confluence 특수 태그로 조정
    let html = marked.parse(markdown) as string;

    // 코드 블록 → Confluence 코드 매크로
    html = html.replace(
        /<pre><code class="language-(\w+)">([\s\S]*?)<\/code><\/pre>/g,
        (_, lang, code) => `
<ac:structured-macro ac:name="code">
  <ac:parameter ac:name="language">${lang}</ac:parameter>
  <ac:plain-text-body><![CDATA[${decode(code)}]]></ac:plain-text-body>
</ac:structured-macro>`
    );

    // 코드 블록 (언어 없음)
    html = html.replace(
        /<pre><code>([\s\S]*?)<\/code><\/pre>/g,
        (_, code) => `
<ac:structured-macro ac:name="code">
  <ac:plain-text-body><![CDATA[${decode(code)}]]></ac:plain-text-body>
</ac:structured-macro>`
    );

    // 인라인 코드
    html = html.replace(/<code>(.*?)<\/code>/g, '<code>$1</code>');

    return html;
}

// Confluence Storage Format → 마크다운 (읽기용)
export function confluenceToMarkdown(storageFormat: string): string {
    let text = storageFormat;

    // 코드 매크로 → 마크다운 코드 블록
    text = text.replace(
        /<ac:structured-macro ac:name="code">[\s\S]*?<ac:parameter ac:name="language">(.*?)<\/ac:parameter>[\s\S]*?<!\[CDATA\[([\s\S]*?)\]\]>[\s\S]*?<\/ac:structured-macro>/g,
        (_, lang, code) => `\`\`\`${lang}\n${code.trim()}\n\`\`\``
    );

    // 기본 HTML 태그 변환
    text = text
        .replace(/<h1[^>]*>(.*?)<\/h1>/g, '# $1')
        .replace(/<h2[^>]*>(.*?)<\/h2>/g, '## $1')
        .replace(/<h3[^>]*>(.*?)<\/h3>/g, '### $1')
        .replace(/<strong[^>]*>(.*?)<\/strong>/g, '**$1**')
        .replace(/<em[^>]*>(.*?)<\/em>/g, '*$1*')
        .replace(/<li[^>]*>(.*?)<\/li>/g, '- $1')
        .replace(/<p[^>]*>(.*?)<\/p>/g, '$1\n\n')
        .replace(/<[^>]+>/g, ''); // 나머지 태그 제거

    return text.trim();
}

function decode(str: string): string {
    return str.replace(/&amp;/g, '&').replace(/&lt;/g, '<').replace(/&gt;/g, '>');
}
```

---

## Step 3: 메시지 처리 핸들러

```typescript
// src/handler.ts
import { Response } from 'express';
import { ConfluenceClient } from './confluence/client';
import { markdownToConfluence, confluenceToMarkdown } from './confluence/formatter';
import { sendSSE, endSSE } from './utils/sse';

const confluence = new ConfluenceClient();

export async function handleConfluenceMessage(
    message: string,
    fileContext: string | null,  // #file: 참조로 넘어온 코드 내용
    res: Response
) {
    try {
        // 1. 문서 검색
        if (isSearchIntent(message)) {
            await handleSearch(message, res);

        // 2. 문서 읽기
        } else if (isReadIntent(message)) {
            await handleRead(message, res);

        // 3. 새 페이지 생성
        } else if (isCreateIntent(message)) {
            await handleCreate(message, fileContext, res);

        // 4. 기존 페이지 수정
        } else if (isUpdateIntent(message)) {
            await handleUpdate(message, fileContext, res);

        } else {
            sendSSE(res, '사용 가능한 명령:\n');
            sendSSE(res, '- `"[검색어]" 찾아줘` / `검색` : 페이지 검색\n');
            sendSSE(res, '- `[페이지명] 내용 보여줘` : 페이지 읽기\n');
            sendSSE(res, '- `[제목]으로 페이지 만들어줘` : 새 페이지 생성\n');
            sendSSE(res, '- `[페이지명] 업데이트해줘` : 기존 페이지 수정\n');
        }

        endSSE(res);

    } catch (error) {
        sendSSE(res, `\n오류: ${error.message}\n`);
        endSSE(res);
    }
}

// --- 검색 ---
async function handleSearch(message: string, res: Response) {
    const query = extractQuery(message);
    sendSSE(res, `"**${query}**" 검색 중...\n\n`);

    const pages = await confluence.searchPages(query);

    if (pages.length === 0) {
        sendSSE(res, '검색 결과가 없습니다.\n');
        return;
    }

    sendSSE(res, `**${pages.length}개** 페이지를 찾았습니다:\n\n`);
    pages.forEach((page, i) => {
        const url = `${process.env.CONFLUENCE_BASE_URL}${page._links.webui}`;
        sendSSE(res, `${i + 1}. [${page.title}](${url}) — ${page.space.name}\n`);
    });
}

// --- 읽기 ---
async function handleRead(message: string, res: Response) {
    const query = extractQuery(message);
    sendSSE(res, `"**${query}**" 페이지 검색 중...\n\n`);

    const pages = await confluence.searchPages(query);
    if (pages.length === 0) {
        sendSSE(res, '페이지를 찾을 수 없습니다.\n');
        return;
    }

    const page = pages[0];
    const fullPage = await confluence.getPage(page.id);
    const markdown = confluenceToMarkdown(fullPage.body.storage.value);
    const url = `${process.env.CONFLUENCE_BASE_URL}${fullPage._links.webui}`;

    sendSSE(res, `## ${fullPage.title}\n`);
    sendSSE(res, `> 스페이스: ${fullPage.space.name} | [Confluence에서 보기](${url})\n\n`);
    sendSSE(res, `---\n\n`);
    sendSSE(res, markdown);
}

// --- 생성 ---
async function handleCreate(message: string, fileContext: string | null, res: Response) {
    const { title, spaceKey } = extractCreateParams(message);
    sendSSE(res, `**"${title}"** 페이지 초안 작성 중...\n\n`);

    // Copilot이 문서 내용을 마크다운으로 생성
    const markdownContent = await generateDocumentContent(message, fileContext, title);

    sendSSE(res, `초안 내용:\n\n---\n\n${markdownContent}\n\n---\n\n`);
    sendSSE(res, `Confluence에 페이지를 생성하는 중...\n`);

    const confluenceContent = markdownToConfluence(markdownContent);
    const page = await confluence.createPage({
        title,
        spaceKey: spaceKey || 'DEV',
        content: confluenceContent,
    });

    const url = `${process.env.CONFLUENCE_BASE_URL}${page._links.webui}`;
    sendSSE(res, `\n✅ 페이지가 생성되었습니다: [${page.title}](${url})\n`);
}

// --- 수정 ---
async function handleUpdate(message: string, fileContext: string | null, res: Response) {
    const query = extractQuery(message);
    sendSSE(res, `"**${query}**" 페이지 찾는 중...\n\n`);

    const pages = await confluence.searchPages(query);
    if (pages.length === 0) {
        sendSSE(res, '페이지를 찾을 수 없습니다.\n');
        return;
    }

    const page = pages[0];
    const fullPage = await confluence.getPage(page.id);
    const existingContent = confluenceToMarkdown(fullPage.body.storage.value);

    sendSSE(res, `"**${fullPage.title}**" 업데이트 내용 생성 중...\n\n`);
    const updatedMarkdown = await generateUpdateContent(message, fileContext, existingContent);

    const confluenceContent = markdownToConfluence(updatedMarkdown);
    const updated = await confluence.updatePage({
        pageId: page.id,
        content: confluenceContent,
    });

    const url = `${process.env.CONFLUENCE_BASE_URL}${updated._links.webui}`;
    sendSSE(res, `\n✅ 페이지가 업데이트되었습니다: [${updated.title}](${url})\n`);
}

// --- 문서 내용 생성 (Claude/GPT API 호출 또는 템플릿) ---
async function generateDocumentContent(
    instruction: string,
    fileContext: string | null,
    title: string
): Promise<string> {
    // 옵션 A: 템플릿 기반 생성
    if (fileContext) {
        return `# ${title}\n\n## 개요\n\n${instruction}\n\n## 코드 참고\n\n\`\`\`\n${fileContext}\n\`\`\`\n\n## 상세 설명\n\n(내용 작성 필요)\n`;
    }
    // 옵션 B: Claude API 연동으로 실제 문서 자동 생성 (아래 섹션 참고)
    return `# ${title}\n\n> 자동 생성된 초안입니다. 내용을 검토 후 수정해주세요.\n\n## 개요\n\n## 주요 내용\n\n## 참고 자료\n`;
}
```

---

## Step 4: Claude API로 고품질 문서 자동 생성

단순 템플릿 대신 **Claude API**를 연동하면 코드/컨텍스트를 분석해서 실제 문서를 생성합니다.

```typescript
// src/confluence/doc-generator.ts
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export async function generateConfluenceDoc(params: {
    instruction: string;
    fileContext?: string;
    existingContent?: string;
    docType: 'new' | 'update';
}): Promise<string> {

    const systemPrompt = `당신은 Confluence 문서를 작성하는 전문가입니다.
마크다운 형식으로 문서를 작성하세요.
- 명확한 제목 계층 구조 사용 (h1, h2, h3)
- 코드가 있으면 코드 블록 사용
- 표가 필요하면 마크다운 표 형식 사용
- 한국어로 작성`;

    const userPrompt = params.docType === 'new'
        ? `다음 지시사항에 따라 Confluence 문서를 작성해줘:\n${params.instruction}
           ${params.fileContext ? `\n\n참고 코드:\n\`\`\`\n${params.fileContext}\n\`\`\`` : ''}`
        : `기존 문서를 다음 지시사항에 따라 업데이트해줘:\n${params.instruction}
           \n\n기존 내용:\n${params.existingContent}
           ${params.fileContext ? `\n\n참고 코드:\n\`\`\`\n${params.fileContext}\n\`\`\`` : ''}`;

    const message = await anthropic.messages.create({
        model: 'claude-sonnet-4-6',
        max_tokens: 4096,
        messages: [{ role: 'user', content: userPrompt }],
        system: systemPrompt,
    });

    return (message.content[0] as { text: string }).text;
}
```

---

## Step 5: 서버 진입점

```typescript
// src/index.ts
import express from 'express';
import { handleConfluenceMessage } from './handler';
import { verifyWebhookSignature } from './utils/auth';

const app = express();
app.use(express.json());

app.post('/agent', verifyWebhookSignature, async (req, res) => {
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');

    const messages = req.body.messages as Array<{ role: string; content: string }>;
    const lastMessage = messages[messages.length - 1].content;

    // #file: 참조로 넘어온 코드 컨텍스트 파싱
    const fileContext = extractFileContext(messages);

    await handleConfluenceMessage(lastMessage, fileContext, res);
    res.end();
});

function extractFileContext(messages: Array<{ role: string; content: string }>): string | null {
    // Copilot이 #file: 참조 내용을 메시지에 포함해서 전달
    const allContent = messages.map(m => m.content).join('\n');
    const match = allContent.match(/```[\s\S]*?```/);
    return match ? match[0] : null;
}

app.listen(3000);
```

---

## 실전 사용 시나리오

### 시나리오 1: 코드 → API 문서 자동 생성
```
// VSCode에서
1. API 파일 열기 (src/routes/user.ts)
2. Copilot Chat에서:

@confluence #file:src/routes/user.ts
이 파일을 보고 API 문서를 DEV 스페이스에 만들어줘
제목은 "User API 명세서"로 해줘
```

### 시나리오 2: PR 내용 → 변경사항 문서 업데이트
```
@confluence #file:CHANGELOG.md
"배포 이력" 페이지에 이번 변경사항 추가해줘
```

### 시나리오 3: 회의 중 실시간 문서 작성
```
@confluence 오늘 논의한 내용으로 회의록 페이지 만들어줘:
- 참석자: 김개발, 이기획, 박디자인
- 결정사항: 로그인 페이지 리뉴얼 2주 안에 완료
- 액션 아이템: 김개발 - 프론트엔드 개발, 이기획 - 요구사항 정리
```

### 시나리오 4: 기존 문서 최신화
```
@confluence "온보딩 가이드" 페이지를 찾아서
환경 설정 섹션에 Node.js 20 설치 방법 추가해줘
```

---

## 패키지 설치

```bash
npm install express axios marked @anthropic-ai/sdk
npm install -D typescript @types/node @types/express ts-node
```

---

## 전체 에이전트 아키텍처

```
VSCode Copilot Chat
        │
        │ @confluence "문서 만들어줘"
        │ + #file: 코드 컨텍스트
        ▼
[Confluence Agent 서버]
        │
        ├── 의도 파악 (검색/읽기/생성/수정)
        │
        ├── Claude API → 문서 내용 생성 (마크다운)
        │
        ├── 마크다운 → Confluence Storage Format 변환
        │
        └── Confluence REST API → 페이지 생성/수정
                │
                ▼
         Confluence 페이지 생성 완료 🎉
         + 링크를 채팅에 반환
```

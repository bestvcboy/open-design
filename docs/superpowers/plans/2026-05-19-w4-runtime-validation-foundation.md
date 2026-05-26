# W4 Runtime Validation Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a reusable runtime validation foundation for `apps/daemon` and apply it to `POST /api/chat` plus all `POST /api/proxy/*/stream` routes with shared `VALIDATION_FAILED` responses.

**Architecture:** Keep runtime request schemas in `packages/contracts` so compile-time and runtime API shape stay aligned. Add one thin daemon-side parser helper that maps schema failures to the shared error envelope, then wire `server.ts` routes to validate before they enter chat/proxy execution or local capability checks.

**Tech Stack:** TypeScript, Zod, Express, Vitest, pnpm workspaces

---

## File Structure

- Modify: `packages/contracts/src/api/comments.ts`
  - Add reusable Zod schemas for preview-comment position, selection kind, and pod members.
- Modify: `packages/contracts/src/api/chat.ts`
  - Add `ChatCommentAttachmentSchema` and `ChatRequestSchema`.
- Modify: `packages/contracts/src/api/proxy.ts`
  - Add `ProxyMessageRoleSchema`, `ProxyMessageSchema`, `ProxyStreamRequestSchema`, and `GoogleProxyStreamRequestSchema`.
- Modify: `packages/contracts/src/index.ts`
  - Re-export the new runtime schemas.
- Create: `packages/contracts/tests/api-validation.test.ts`
  - Lock request-schema behavior for chat and proxy contracts.
- Create: `apps/daemon/src/validation.ts`
  - Provide one focused helper that turns `schema.safeParse()` results into `{ ok, data }` or shared validation details.
- Create: `apps/daemon/tests/validation.test.ts`
  - Lock issue-path mapping and validation detail shape.
- Modify: `apps/daemon/src/server.ts`
  - Import contracts schemas and daemon validation helper, then validate `chat` and proxy route bodies before execution.
- Modify: `apps/daemon/tests/chat-route.test.ts`
  - Add route-level proof for malformed chat payloads returning `VALIDATION_FAILED`.
- Modify: `apps/daemon/tests/proxy-routes.test.ts`
  - Add route-level proof for malformed proxy payloads returning `VALIDATION_FAILED` while preserving existing happy-path proxy behavior.

## Task 1: Add chat-side runtime schemas to contracts

**Files:**
- Create: `packages/contracts/tests/api-validation.test.ts`
- Modify: `packages/contracts/src/api/comments.ts`
- Modify: `packages/contracts/src/api/chat.ts`
- Modify: `packages/contracts/src/index.ts`

- [ ] **Step 1: Write the failing contracts test for chat request parsing**

```ts
import { describe, expect, it } from 'vitest';

import {
  ChatRequestSchema,
  PreviewCommentMemberSchema,
  PreviewCommentPositionSchema,
} from '../src/index';

describe('ChatRequestSchema', () => {
  it('accepts a valid chat payload with nested comment attachments', () => {
    const parsed = ChatRequestSchema.safeParse({
      agentId: 'claude',
      message: 'tighten the hero copy',
      projectId: 'project-1',
      commentAttachments: [
        {
          id: 'comment-1',
          order: 1,
          filePath: 'src/index.html',
          elementId: 'hero-title',
          selector: '#hero-title',
          label: 'Hero title',
          comment: 'Make this headline punchier.',
          currentText: 'Build faster',
          pagePosition: { x: 12, y: 24, width: 320, height: 72 },
          htmlHint: '<h1 id="hero-title">Build faster</h1>',
          selectionKind: 'pod',
          memberCount: 1,
          podMembers: [
            {
              elementId: 'hero-title',
              selector: '#hero-title',
              label: 'Hero title',
              text: 'Build faster',
              position: { x: 12, y: 24, width: 320, height: 72 },
              htmlHint: '<h1 id="hero-title">Build faster</h1>',
            },
          ],
          source: 'saved-comment',
        },
      ],
    });

    expect(parsed.success).toBe(true);
  });

  it('rejects malformed nested comment attachments with stable field paths', () => {
    const parsed = ChatRequestSchema.safeParse({
      agentId: 'claude',
      message: 'hello',
      commentAttachments: [
        {
          id: 'comment-1',
          order: 'first',
          filePath: 'src/index.html',
        },
      ],
    });

    expect(parsed.success).toBe(false);
    expect(parsed.error.issues.map((issue) => issue.path.join('.'))).toContain(
      'commentAttachments.0.order',
    );
  });
});

describe('preview comment schemas', () => {
  it('accepts valid preview comment position and member payloads', () => {
    expect(
      PreviewCommentPositionSchema.safeParse({
        x: 1,
        y: 2,
        width: 320,
        height: 180,
      }).success,
    ).toBe(true);

    expect(
      PreviewCommentMemberSchema.safeParse({
        elementId: 'cta',
        selector: '#cta',
        label: 'CTA',
        text: 'Start now',
        position: { x: 1, y: 2, width: 3, height: 4 },
        htmlHint: '<button id="cta">Start now</button>',
      }).success,
    ).toBe(true);
  });
});
```

- [ ] **Step 2: Run the contracts test to verify it fails**

Run:

```bash
pnpm --filter @open-design/contracts test -- tests/api-validation.test.ts
```

Expected: FAIL because `ChatRequestSchema`, `PreviewCommentPositionSchema`, and `PreviewCommentMemberSchema` do not exist yet.

- [ ] **Step 3: Add reusable comment and chat request schemas**

Update `packages/contracts/src/api/comments.ts`:

```ts
import { z } from 'zod';
import type { OkResponse } from '../common';

export const PreviewCommentPositionSchema = z.object({
  x: z.number().finite(),
  y: z.number().finite(),
  width: z.number().finite(),
  height: z.number().finite(),
});
export type PreviewCommentPosition = z.infer<typeof PreviewCommentPositionSchema>;

export const PreviewCommentSelectionKindSchema = z.enum(['element', 'pod']);
export type PreviewCommentSelectionKind = z.infer<
  typeof PreviewCommentSelectionKindSchema
>;

export const PreviewCommentMemberSchema = z.object({
  elementId: z.string().trim().min(1),
  selector: z.string().trim().min(1),
  label: z.string(),
  text: z.string(),
  position: PreviewCommentPositionSchema,
  htmlHint: z.string(),
});
export type PreviewCommentMember = z.infer<typeof PreviewCommentMemberSchema>;
```

Update `packages/contracts/src/api/chat.ts`:

```ts
import { z } from 'zod';
import type { ProjectFile } from './files';
import type {
  PreviewCommentMember,
  PreviewCommentPosition,
  PreviewCommentSelectionKind,
} from './comments';
import {
  PreviewCommentMemberSchema,
  PreviewCommentPositionSchema,
  PreviewCommentSelectionKindSchema,
} from './comments';

const NullableIdSchema = z.string().trim().min(1).nullable().optional();

export const ChatCommentAttachmentSchema = z.object({
  id: z.string().trim().min(1),
  order: z.number().int().min(1),
  filePath: z.string().trim().min(1),
  elementId: z.string().trim().min(1),
  selector: z.string().trim().min(1),
  label: z.string(),
  comment: z.string().trim().min(1),
  currentText: z.string(),
  pagePosition: PreviewCommentPositionSchema,
  htmlHint: z.string(),
  selectionKind: PreviewCommentSelectionKindSchema.optional(),
  memberCount: z.number().int().min(0).optional(),
  podMembers: z.array(PreviewCommentMemberSchema).optional(),
  source: z.enum(['saved-comment', 'board-batch']).optional(),
});
export type ChatCommentAttachment = z.infer<typeof ChatCommentAttachmentSchema>;

export const ChatRequestSchema = z.object({
  agentId: z.string().trim().min(1),
  message: z.string().trim().min(1),
  systemPrompt: z.string().optional(),
  projectId: NullableIdSchema,
  conversationId: NullableIdSchema,
  assistantMessageId: NullableIdSchema,
  clientRequestId: NullableIdSchema,
  skillId: NullableIdSchema,
  designSystemId: NullableIdSchema,
  attachments: z.array(z.string().trim().min(1)).optional(),
  commentAttachments: z.array(ChatCommentAttachmentSchema).optional(),
  model: NullableIdSchema,
  reasoning: z.string().nullable().optional(),
});
```

Update `packages/contracts/src/index.ts`:

```ts
export * from './api/chat';
export * from './api/comments';
```

- [ ] **Step 4: Run contracts tests and typecheck**

Run:

```bash
pnpm --filter @open-design/contracts test -- tests/api-validation.test.ts
pnpm --filter @open-design/contracts typecheck
```

Expected:

- the new chat contracts tests PASS
- contracts package typecheck PASS

- [ ] **Step 5: Commit the chat contracts slice**

```bash
git -C G:\AUDHOUSE\HOUSEDRAW\open-design add -- packages/contracts/src/api/comments.ts packages/contracts/src/api/chat.ts packages/contracts/src/index.ts packages/contracts/tests/api-validation.test.ts
git -C G:\AUDHOUSE\HOUSEDRAW\open-design commit -m "feat(contracts): add chat request schemas"
```

## Task 2: Add proxy runtime schemas to contracts

**Files:**
- Modify: `packages/contracts/tests/api-validation.test.ts`
- Modify: `packages/contracts/src/api/proxy.ts`
- Modify: `packages/contracts/src/index.ts`

- [ ] **Step 1: Extend the contracts test with proxy schema coverage**

Add to `packages/contracts/tests/api-validation.test.ts`:

```ts
import {
  GoogleProxyStreamRequestSchema,
  ProxyStreamRequestSchema,
} from '../src/index';

describe('ProxyStreamRequestSchema', () => {
  it('accepts a valid OpenAI-compatible proxy payload', () => {
    const parsed = ProxyStreamRequestSchema.safeParse({
      baseUrl: 'https://api.example.com/v1',
      apiKey: 'sk-test',
      model: 'gpt-5-mini',
      messages: [{ role: 'user', content: 'hello' }],
      maxTokens: 1024,
    });

    expect(parsed.success).toBe(true);
  });

  it('rejects malformed proxy messages with stable field paths', () => {
    const parsed = ProxyStreamRequestSchema.safeParse({
      baseUrl: 'https://api.example.com/v1',
      apiKey: 'sk-test',
      model: 'gpt-5-mini',
      messages: [{ role: 'narrator', content: '' }],
    });

    expect(parsed.success).toBe(false);
    expect(parsed.error.issues.map((issue) => issue.path.join('.'))).toEqual(
      expect.arrayContaining(['messages.0.role', 'messages.0.content']),
    );
  });

  it('allows Google proxy requests to omit baseUrl', () => {
    const parsed = GoogleProxyStreamRequestSchema.safeParse({
      apiKey: 'google-key',
      model: 'gemini-2.0-flash',
      messages: [{ role: 'user', content: 'hello' }],
    });

    expect(parsed.success).toBe(true);
  });
});
```

- [ ] **Step 2: Run the contracts test to verify proxy schema coverage fails**

Run:

```bash
pnpm --filter @open-design/contracts test -- tests/api-validation.test.ts
```

Expected: FAIL because the proxy schemas are not exported yet.

- [ ] **Step 3: Implement proxy request schemas**

Update `packages/contracts/src/api/proxy.ts`:

```ts
import { z } from 'zod';

export const ProxyMessageRoleSchema = z.enum([
  'system',
  'user',
  'assistant',
  'tool',
]);
export type ProxyMessageRole = z.infer<typeof ProxyMessageRoleSchema>;

export const ProxyMessageSchema = z.object({
  role: ProxyMessageRoleSchema,
  content: z.string().trim().min(1),
});
export type ProxyMessage = z.infer<typeof ProxyMessageSchema>;

const ProxyRequestBaseFields = {
  apiKey: z.string().trim().min(1),
  model: z.string().trim().min(1),
  systemPrompt: z.string().optional(),
  messages: z.array(ProxyMessageSchema).min(1),
  maxTokens: z.number().int().positive().optional(),
  apiVersion: z.string().trim().min(1).optional(),
};

export const ProxyStreamRequestSchema = z.object({
  baseUrl: z.string().trim().url(),
  ...ProxyRequestBaseFields,
});
export type ProxyStreamRequest = z.infer<typeof ProxyStreamRequestSchema>;

export const GoogleProxyStreamRequestSchema = z.object({
  baseUrl: z.string().trim().url().optional(),
  ...ProxyRequestBaseFields,
});
```

Update `packages/contracts/src/index.ts`:

```ts
export * from './api/proxy';
```

- [ ] **Step 4: Re-run contracts tests and typecheck**

Run:

```bash
pnpm --filter @open-design/contracts test -- tests/api-validation.test.ts
pnpm --filter @open-design/contracts typecheck
```

Expected:

- proxy schema tests PASS
- the combined contracts schema test file PASS
- contracts typecheck PASS

- [ ] **Step 5: Commit the proxy contracts slice**

```bash
git -C G:\AUDHOUSE\HOUSEDRAW\open-design add -- packages/contracts/src/api/proxy.ts packages/contracts/src/index.ts packages/contracts/tests/api-validation.test.ts
git -C G:\AUDHOUSE\HOUSEDRAW\open-design commit -m "feat(contracts): add proxy request schemas"
```

## Task 3: Add a daemon-side validation helper

**Files:**
- Create: `apps/daemon/src/validation.ts`
- Create: `apps/daemon/tests/validation.test.ts`

- [ ] **Step 1: Write the failing daemon validation helper test**

Create `apps/daemon/tests/validation.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { z } from 'zod';

import { parseRequest, zodPathToString } from '../src/validation.js';

describe('zodPathToString', () => {
  it('renders nested paths using dot + bracket notation', () => {
    expect(zodPathToString(['commentAttachments', 0, 'order'])).toBe(
      'commentAttachments[0].order',
    );
  });
});

describe('parseRequest', () => {
  const schema = z.object({
    message: z.string().trim().min(1),
    nested: z.object({
      count: z.number().int(),
    }),
  });

  it('returns parsed data when validation succeeds', () => {
    expect(
      parseRequest(schema, {
        message: 'hello',
        nested: { count: 3 },
      }),
    ).toEqual({
      ok: true,
      data: { message: 'hello', nested: { count: 3 } },
    });
  });

  it('returns shared validation details when validation fails', () => {
    const result = parseRequest(schema, {
      message: '',
      nested: { count: 'nope' },
    });

    expect(result.ok).toBe(false);
    if (result.ok) throw new Error('expected validation failure');

    expect(result.details).toEqual({
      kind: 'validation',
      issues: expect.arrayContaining([
        expect.objectContaining({ path: 'message' }),
        expect.objectContaining({ path: 'nested.count' }),
      ]),
    });
  });
});
```

- [ ] **Step 2: Run the daemon helper test to verify it fails**

Run:

```bash
pnpm --filter @open-design/daemon test -- tests/validation.test.ts
```

Expected: FAIL because `apps/daemon/src/validation.ts` does not exist yet.

- [ ] **Step 3: Implement the validation helper**

Create `apps/daemon/src/validation.ts`:

```ts
import type { ApiValidationErrorDetails, ApiValidationIssue } from '@open-design/contracts';
import { z, type ZodTypeAny } from 'zod';

export function zodPathToString(path: (string | number)[]): string {
  return path.reduce((acc, part) => {
    if (typeof part === 'number') return `${acc}[${part}]`;
    return acc ? `${acc}.${part}` : part;
  }, '');
}

export function zodIssuesToValidationDetails(
  error: z.ZodError,
): ApiValidationErrorDetails {
  const issues: ApiValidationIssue[] = error.issues.map((issue) => ({
    path: zodPathToString(issue.path),
    message: issue.message,
    code: issue.code,
  }));

  return {
    kind: 'validation',
    issues,
  };
}

export function parseRequest<S extends ZodTypeAny>(
  schema: S,
  input: unknown,
): { ok: true; data: z.infer<S> } | { ok: false; details: ApiValidationErrorDetails } {
  const parsed = schema.safeParse(input);
  if (parsed.success) {
    return { ok: true, data: parsed.data };
  }

  return {
    ok: false,
    details: zodIssuesToValidationDetails(parsed.error),
  };
}
```

- [ ] **Step 4: Re-run the daemon helper test**

Run:

```bash
pnpm --filter @open-design/daemon test -- tests/validation.test.ts
pnpm --filter @open-design/daemon typecheck
```

Expected:

- validation helper tests PASS
- daemon typecheck PASS

- [ ] **Step 5: Commit the validation helper slice**

```bash
git -C G:\AUDHOUSE\HOUSEDRAW\open-design add -- apps/daemon/src/validation.ts apps/daemon/tests/validation.test.ts
git -C G:\AUDHOUSE\HOUSEDRAW\open-design commit -m "feat(daemon): add request validation helper"
```

## Task 4: Validate `/api/chat` before creating a run

**Files:**
- Modify: `apps/daemon/src/server.ts`
- Modify: `apps/daemon/tests/chat-route.test.ts`

- [ ] **Step 1: Add failing route tests for malformed chat payloads**

Add to `apps/daemon/tests/chat-route.test.ts`:

```ts
  it('returns VALIDATION_FAILED for malformed chat payloads before creating a run', async () => {
    const response = await fetch(`${baseUrl}/api/chat`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        agentId: 'claude',
        commentAttachments: [
          {
            id: 'comment-1',
            order: 'first',
            filePath: 'src/index.html',
          },
        ],
      }),
    });

    expect(response.status).toBe(400);
    await expect(response.json()).resolves.toMatchObject({
      error: {
        code: 'VALIDATION_FAILED',
        details: {
          kind: 'validation',
          issues: expect.arrayContaining([
            expect.objectContaining({ path: 'message' }),
            expect.objectContaining({ path: 'commentAttachments[0].order' }),
          ]),
        },
      },
    });
  });
```

- [ ] **Step 2: Run the chat route test to verify it fails**

Run:

```bash
pnpm --filter @open-design/daemon test -- tests/chat-route.test.ts
```

Expected: FAIL because the route currently creates a run immediately and does not return a shared validation envelope.

- [ ] **Step 3: Wire chat route validation into `server.ts`**

Update imports near the top of `apps/daemon/src/server.ts`:

```ts
import { ChatRequestSchema } from '@open-design/contracts';
import { parseRequest } from './validation.js';
```

Update the route:

```ts
  app.post('/api/chat', (req, res) => {
    const parsedBody = parseRequest(ChatRequestSchema, req.body);
    if (!parsedBody.ok) {
      return sendApiError(
        res,
        400,
        'VALIDATION_FAILED',
        'Invalid request payload',
        { details: parsedBody.details },
      );
    }

    const run = design.runs.create();
    design.runs.stream(run, req, res);
    design.runs.start(run, () => startChatRun(parsedBody.data, run));
  });
```

- [ ] **Step 4: Re-run chat route tests**

Run:

```bash
pnpm --filter @open-design/daemon test -- tests/chat-route.test.ts
pnpm --filter @open-design/daemon typecheck
```

Expected:

- chat route tests PASS
- daemon typecheck PASS

- [ ] **Step 5: Commit the chat route slice**

```bash
git -C G:\AUDHOUSE\HOUSEDRAW\open-design add -- apps/daemon/src/server.ts apps/daemon/tests/chat-route.test.ts
git -C G:\AUDHOUSE\HOUSEDRAW\open-design commit -m "feat(daemon): validate chat request bodies"
```

## Task 5: Validate all proxy stream routes before provider-specific security checks

**Files:**
- Modify: `apps/daemon/src/server.ts`
- Modify: `apps/daemon/tests/proxy-routes.test.ts`

- [ ] **Step 1: Add failing proxy route validation tests**

Add to `apps/daemon/tests/proxy-routes.test.ts`:

```ts
  it('returns VALIDATION_FAILED for malformed OpenAI-compatible proxy payloads', async () => {
    const res = await realFetch(`${baseUrl}/api/proxy/openai/stream`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({
        baseUrl: 'https://api.example.com/v1',
        apiKey: 'sk-test',
        messages: [{ role: 'narrator', content: '' }],
      }),
    });

    expect(res.status).toBe(400);
    await expect(res.json()).resolves.toMatchObject({
      error: {
        code: 'VALIDATION_FAILED',
        details: {
          kind: 'validation',
          issues: expect.arrayContaining([
            expect.objectContaining({ path: 'model' }),
            expect.objectContaining({ path: 'messages[0].role' }),
            expect.objectContaining({ path: 'messages[0].content' }),
          ]),
        },
      },
    });
  });

  it('returns VALIDATION_FAILED when Google proxy payload omits apiKey', async () => {
    const res = await realFetch(`${baseUrl}/api/proxy/google/stream`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({
        model: 'gemini-2.0-flash',
        messages: [{ role: 'user', content: 'hello' }],
      }),
    });

    expect(res.status).toBe(400);
    await expect(res.json()).resolves.toMatchObject({
      error: {
        code: 'VALIDATION_FAILED',
        details: {
          kind: 'validation',
          issues: expect.arrayContaining([
            expect.objectContaining({ path: 'apiKey' }),
          ]),
        },
      },
    });
  });
```

- [ ] **Step 2: Run proxy route tests to verify they fail**

Run:

```bash
pnpm --filter @open-design/daemon test -- tests/proxy-routes.test.ts
```

Expected: FAIL because the proxy routes still return route-local `BAD_REQUEST` checks instead of shared validation errors.

- [ ] **Step 3: Wire proxy schemas into all proxy routes**

Update imports near the top of `apps/daemon/src/server.ts`:

```ts
import {
  GoogleProxyStreamRequestSchema,
  ProxyStreamRequestSchema,
} from '@open-design/contracts';
import { parseRequest } from './validation.js';
```

Update Anthropic/OpenAI/Azure routes:

```ts
    const parsedBody = parseRequest(ProxyStreamRequestSchema, req.body);
    if (!parsedBody.ok) {
      return sendApiError(
        res,
        400,
        'VALIDATION_FAILED',
        'Invalid request payload',
        { details: parsedBody.details },
      );
    }

    const { baseUrl, apiKey, model, systemPrompt, messages, maxTokens } =
      parsedBody.data;
```

Update Google route:

```ts
    const parsedBody = parseRequest(GoogleProxyStreamRequestSchema, req.body);
    if (!parsedBody.ok) {
      return sendApiError(
        res,
        400,
        'VALIDATION_FAILED',
        'Invalid request payload',
        { details: parsedBody.details },
      );
    }

    const {
      baseUrl,
      apiKey,
      model,
      systemPrompt,
      messages,
      maxTokens,
    } = parsedBody.data;
    const effectiveBaseUrl =
      baseUrl || 'https://generativelanguage.googleapis.com';
```

Do not remove `validateExternalApiBaseUrl()`. Keep it after schema parsing so malformed shapes return `VALIDATION_FAILED` and valid-but-forbidden targets still return `FORBIDDEN`.

- [ ] **Step 4: Re-run proxy route tests and daemon typecheck**

Run:

```bash
pnpm --filter @open-design/daemon test -- tests/proxy-routes.test.ts
pnpm --filter @open-design/daemon typecheck
```

Expected:

- proxy route tests PASS
- existing proxy happy-path tests still PASS
- daemon typecheck PASS

- [ ] **Step 5: Commit the proxy route slice**

```bash
git -C G:\AUDHOUSE\HOUSEDRAW\open-design add -- apps/daemon/src/server.ts apps/daemon/tests/proxy-routes.test.ts
git -C G:\AUDHOUSE\HOUSEDRAW\open-design commit -m "feat(daemon): validate proxy request bodies"
```

## Task 6: Run full verification for the W4 first slice

**Files:**
- Modify: none

- [ ] **Step 1: Run the focused contracts tests**

Run:

```bash
pnpm --filter @open-design/contracts test -- tests/api-validation.test.ts
```

Expected: PASS

- [ ] **Step 2: Run the focused daemon tests**

Run:

```bash
pnpm --filter @open-design/daemon test -- tests/validation.test.ts tests/chat-route.test.ts tests/proxy-routes.test.ts
```

Expected: PASS

- [ ] **Step 3: Run workspace-required checks**

Run:

```bash
pnpm guard
pnpm typecheck
```

Expected: PASS

- [ ] **Step 4: Inspect working tree before handoff**

Run:

```bash
git -C G:\AUDHOUSE\HOUSEDRAW\open-design --no-pager status --short
```

Expected: only the planned W4 validation files remain changed.

- [ ] **Step 5: Inspect the final commit stack**

Run:

```bash
git -C G:\AUDHOUSE\HOUSEDRAW\open-design --no-pager log --oneline -5
```

Expected: the recent history shows the contracts, helper, chat-route, and proxy-route commits created in Tasks 1-5.

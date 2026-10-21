---
name: security-boundaries
description: Use when changing LobeHub API routes, authentication or resource access, machine callbacks or jobs, outbound URLs, privileged desktop/device operations, or untrusted content and files.
---

# Security Boundaries

Read this alongside the skill for the feature you are building. Apply the relevant boundary; unrelated tasks do not need every section.

## Before coding

Identify the actor, who can change each input or setting, the resource or destination it selects, and the legitimate request that must still work. A value stored in a database or repository is not automatically trusted: users, agents, integrations, or imported projects may have supplied it. Find a protected sibling path, then check whether its protection also covers the new route, child operation, redirect, fallback, or later job. Do not copy an existing path without checking its boundary.

## Routes and resource access

- Choose an explicit caller mode for every new tRPC, Next, OpenAPI, or Hono route. Public routes may be intentional, but do not assume the Next proxy or a procedure name authenticates an API route. Check token purpose, audience, and API-key scope where those credentials are accepted.
- Treat `X-Workspace-Id` and resource IDs from a request as selectors, not authority. For workspace-capable operations, verify current membership or permission from database state. `buildWorkspaceWhere` scopes rows; it does not prove the caller belongs to the selected workspace.
- Scope reads and writes to the effective target. Child records, shares, copies, links, and transfers may need checks on both the source and destination. Use `packages/database/src/utils/workspace.ts` and the `workspacePermission` and `resourcePermission` services as appropriate; inspect what each helper actually enforces.
- Accept only fields the caller may edit. Do not let request data replace ownership, workspace, visibility, or credential fields through a broad update schema.

## Outbound connections

- Treat a destination influenced by a user, agent, tenant setting, remote response, or redirect as untrusted. For buffered server requests, use `ssrfSafeFetch` from `packages/ssrf-safe-fetch/` with a timeout and response-size limit. It buffers the body; streaming and SDK transports need equivalent protection at connection time, not a blind substitution.
- Apply the destination policy to every redirect, retry, and fallback. Check the actual resolved destination before sending credentials. `apps/server/src/services/bot/platforms/publicUrlFetch.ts` illustrates DNS and redirect handling for a different transport shape.
- Preserve private destinations explicitly permitted by an operator. Do not silently broaden that exception or override the operator's policy per request.

## Callbacks and later work

- An externally reachable webhook or job receiver must verify its caller. A missing signing secret must disable a route that requires signatures, not silently skip verification. Verify the raw request, and address timestamp or delivery-ID replay where the provider supports it.
- Resolve the acting user or workspace from verified provider identity or a scoped database record, not `userId` or `workspaceId` in a callback body. When a job runs later, reload its target under the expected owner/workspace and recheck its current state before acting.
- Bind OAuth callback state to the initiating user and consume it once. Build internal callback destinations from trusted server configuration, not a request-supplied base URL.

## Local execution and untrusted content

- For privileged desktop IPC or device actions, validate arguments and the caller at runtime, authorize the selected device, and enforce file roots after path resolution. Do not build shell commands from untrusted strings or grant a token or approval bypass because command text matches a prefix. Follow `executeAuthorizedDeviceToolCall` for server-to-device calls.
- Parse untrusted frontmatter with `safeMatterOptions` from `apps/server/src/utils/`. Keep executable HTML/SVG previews isolated from the app origin; do not combine script execution with `allow-same-origin`. Bound upload and decompression sizes before processing user files.

## Verification

Test the hostile or unauthorized path and the intended legitimate path for the boundary changed. Include a foreign resource, child operation, private-address redirect, unsigned callback, or later job when relevant. A test that only asserts a helper was called does not establish the boundary.

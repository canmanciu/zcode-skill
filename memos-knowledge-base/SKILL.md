---
name: memos-knowledge-base
description: Read, search, create, and update notes/memos on note.canmanciu.top (Memos v0.31.0) via the REST API using a Bearer token. Use when the user asks to (1) recall / search / summarize something from their notes or knowledge base, (2) save / jot / record a new note or memo, (3) publish or draft a blog post (PUBLIC memo), (4) update or delete an existing memo. Do NOT use for the demo site (demo.canmanciu.top is SSH-only, not API-driven) or for sale.goibuy.cn.
---

# Memos Knowledge Base Skill

## When to use

Trigger this skill when the user says things like:

- "记一下…" / "save this note…" / "记到笔记里" → create memo (default PRIVATE)
- "我之前写过…吗" / "search my notes for…" / "查一下我笔记里关于 X 的内容" → search & read
- "发一篇博客" / "publish this as a blog post" / "把这个写成公开文章" → create PUBLIC memo
- "更新那条…" / "edit my note about…" → PATCH memo
- "删掉…" / "delete the memo…" → DELETE memo
- "总结一下我最近关于 X 的笔记" → list + synthesize

Do NOT trigger for:
- demo.canmanciu.top operations (that's SSH-only, no API)
- sale.goibuy.cn operations
- Generic chat / coding questions that don't touch the notes system

## Setup (one-time)

### 推荐方式：`~/.memos.conf`（持久化，与工具无关）

```bash
echo 'MEMOS_TOKEN="<your-token>"' > ~/.memos.conf
chmod 600 ~/.memos.conf
```

### 备选方式：`~/.zshrc` 导出环境变量

```bash
echo 'export MEMOS_TOKEN="<your-token>"' >> ~/.zshrc
```

ZCode 的 Bash 工具会从用户 shell profile 初始化，export 后即可直接使用 `$MEMOS_TOKEN`。

Token 获取：登录 `https://note.canmanciu.top` → Settings → Access Tokens → 创建。

### 执行时 token 获取逻辑

```
1. 检查 $MEMOS_TOKEN 环境变量（~/.zshrc export 的情况）
2. 若为空，尝试 source ~/.memos.conf
3. 若仍为空，停止并提示用户配置
```

If `$MEMOS_TOKEN` is empty after both sources, **stop and ask the user to set it** — do not attempt anonymous calls for write operations (they will 401).

## Endpoint & auth

- **Base URL**: `https://note.canmanciu.top/api/v1`
- **Auth header**: `Authorization: Bearer $MEMOS_TOKEN`
- **Content-Type**: `application/json` for all write calls
- **Resource name format**: `memos/{id}` (e.g. `memos/42`), `users/{id}` (e.g. `users/1`)

## Visibility decision tree (CRITICAL)

Memos uses one system for both notes and blog posts — visibility is the only difference.

| User intent | visibility |
|---|---|
| "记一下" / "save" / "jot" / "note to self" / private thought | `PRIVATE` (default) |
| "发博客" / "publish" / "blog post" / "公开" / "share publicly" | `PUBLIC` |
| "给协作伙伴看" / "shared with logged-in users" | `PROTECTED` |
| User doesn't specify | `PRIVATE` — never assume PUBLIC; publishing is reversible but a leaked private note is not |

**Default to PRIVATE when unclear.** Always confirm before publishing: "这条要发到博客（PUBLIC）吗？还是只存私有笔记？"

## API call patterns

All examples use `curl` + `jq`. The token must be in `$MEMOS_TOKEN`.

### 1. List recent notes (default 50)

```bash
curl -sS -H "Authorization: Bearer $MEMOS_TOKEN" \
  "https://note.canmanciu.top/api/v1/memos?pageSize=50" | jq .
```

Response shape:
```json
{
  "memos": [
    {
      "name": "memos/1",
      "uid": "doMojVUBQWjkrabmu9Dhpc",
      "content": "#linux 备忘：nginx reload 前先 `nginx -t`",
      "visibility": "PRIVATE",
      "createTime": "2026-07-20T09:04:19Z",
      "updateTime": "2026-07-20T09:04:19Z",
      "creator": "users/1",
      ...
    }
  ],
  "nextPageToken": ""
}
```

### 2. Filter by tag / visibility / content

Filter uses CEL syntax, URL-encoded:

```bash
# By tag
curl -sS -H "Authorization: Bearer $MEMOS_TOKEN" \
  "https://note.canmanciu.top/api/v1/memos?filter=$(jq -rR 'urlencode' <<< 'tag == "linux"')"

# Public only
curl -sS -H "Authorization: Bearer $MEMOS_TOKEN" \
  "https://note.canmanciu.top/api/v1/memos?filter=$(jq -rR 'urlencode' <<< 'visibility == "PUBLIC"')"

# Combined
curl -sS -H "Authorization: Bearer $MEMOS_TOKEN" \
  "https://note.canmanciu.top/api/v1/memos?filter=$(jq -rR 'urlencode' <<< 'tag == "linux" && visibility == "PUBLIC"')"
```

### 3. Full-text search (v0.29.1)

```bash
curl -sS -H "Authorization: Bearer $MEMOS_TOKEN" \
  "https://note.canmanciu.top/api/v1/memos?filter=$(jq -rR 'urlencode' <<< 'content.search("nginx")')"
```

If `content.search` returns an error or empty, fall back to listing all memos and grepping locally:

```bash
curl -sS -H "Authorization: Bearer $MEMOS_TOKEN" \
  "https://note.canmanciu.top/api/v1/memos?pageSize=200" \
  | jq -r '.memos[] | "## \(.name) [\(.visibility)]\n\(.content)\n"'
```

### 4. Create a memo (PRIVATE note)

Content length limit: instance setting `MEMO_RELATED.content_length_limit` (currently 131072 bytes, **counted in UTF-8 bytes** — Chinese ≈ 3 bytes/char; floor is 8192, adjustable via `PATCH /api/v1/instance/settings/MEMO_RELATED?updateMask=memo_related_setting` — note: PATCH replaces the whole `memoRelatedSetting` block, include `reactions`/`enableDoubleClickEdit` or they get wiped). Long articles fit in a single memo; no need to split.

```bash
curl -sS -X POST -H "Authorization: Bearer $MEMOS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg c "$CONTENT" '{content: $c, visibility: "PRIVATE"}')" \
  "https://note.canmanciu.top/api/v1/memos"
```

### 4b. Attach a file to a memo (v0.31 attachments API)

Upload is JSON (no multipart): `POST /api/v1/attachments` with a **flat** body (gRPC-gateway `body:"attachment"` unwraps it — do NOT nest under `"attachment"`), `content` is base64:

```bash
B64=$(base64 -w0 file.pdf)
curl -sS -X POST -H "Authorization: Bearer $MEMOS_TOKEN" -H "Content-Type: application/json" \
  -d "{\"filename\":\"file.pdf\",\"type\":\"application/pdf\",\"content\":\"$B64\",\"memo\":\"memos/42\"}" \
  "https://note.canmanciu.top/api/v1/attachments"
```

- The `memo` field at create often does NOT stick — always follow up with `PATCH /api/v1/attachments/{name}?updateMask=memo` body `{"name":"attachments/{id}","memo":"memos/42"}`. PATCH occasionally returns 200 but silently drops the link — verify via `GET /api/v1/memos/42` and retry/recreate if `attachments` is missing.
- Memos renders Markdown only: raw HTML in memo content is escaped to text, CSS/JS never runs. For styled documents, attach the original file.

### 5. Publish a blog post (PUBLIC memo)

```bash
curl -sS -X POST -H "Authorization: Bearer $MEMOS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg c "$CONTENT" '{content: $c, visibility: "PUBLIC"}')" \
  "https://note.canmanciu.top/api/v1/memos"
```

After publishing, return the public URL to the user:
- Public list: `https://note.canmanciu.top/m/{uid}`
- Direct share: `https://note.canmanciu.top/m/{uid}` (same path; visibility controls who can read)

The `uid` comes from the response field `uid` (a short string like `doMojVUBQWjkrabmu9Dhpc`).

### 6. Update a memo

```bash
# Patch content and/or visibility
curl -sS -X PATCH -H "Authorization: Bearer $MEMOS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg c "$NEW_CONTENT" '{content: $c}')" \
  "https://note.canmanciu.top/api/v1/memos/memos/42"
```

Useful patches:
- Change visibility: `{"visibility": "PUBLIC"}` (private → blog)
- Fix typo: `{"content": "corrected text"}`
- Both: `{"content": "...", "visibility": "PRIVATE"}` (blog → private)

### 7. Delete a memo

```bash
curl -sS -X DELETE -H "Authorization: Bearer $MEMOS_TOKEN" \
  "https://note.canmanciu.top/api/v1/memos/memos/42"
```

**Always confirm before deleting** — Memos has no trash/undo in v0.29.1.

## Tag conventions

Memos parses `#tag` from content automatically. Recommend the user adopt a small tag taxonomy, e.g.:

- `#linux` `#nginx` `#postgres` — technical
- `#idea` `#todo` `#draft` — workflow
- `#blog` `#draft` — for blog pipeline (draft PRIVATE first, then flip to PUBLIC)
- `#ref` — bookmarks / reference links

When creating a memo from a conversation, **auto-add tags** based on topic if the user didn't specify. Example: a note about nginx reload → prepend `#linux #nginx ` to the content.

## Workflow patterns

### Pattern A: "save this" (most common)

1. Extract the salient content from the conversation (don't dump the whole transcript).
2. Add 1–3 relevant `#tags` at the top if not already present.
3. Default `visibility=PRIVATE`.
4. POST, return the `memos/{id}` and a one-line summary back to the user.

### Pattern B: "search my notes for X"

1. Try `content.search("X")` filter first.
2. If empty / errored, list all memos (pageSize=200) and grep locally with `jq`.
3. Synthesize an answer; cite each source as `memos/{id}` + first line of content.
4. Never claim "you haven't written about X" if the search filter failed — only claim that based on a full local grep fallback.

### Pattern C: "publish this as a blog post"

1. Draft the post in clean Markdown (title as `# H1`, then body).
2. **Ask the user to confirm `PUBLIC`** before posting — never auto-publish.
3. POST with `visibility=PUBLIC`.
4. Return the public URL `https://note.canmanciu.top/m/{uid}`.

### Pattern D: "turn my note #N into a blog post"

1. GET `memos/{N}`.
2. Show the user the current content.
3. On confirm, PATCH `visibility` to `PUBLIC` (optionally also edit content).
4. Return the public URL.

## Error handling

| HTTP | Meaning | What to do |
|---|---|---|
| 401 | token missing/invalid | Stop, ask user to configure `MEMOS_TOKEN` (see Setup) |
| 403 | token valid but no permission (e.g. editing other user's memo) | Tell user, don't retry |
| 404 | memo doesn't exist OR is PRIVATE/PROTECTED and not yours | Tell user "not found" |
| 400 | bad request (bad filter syntax, missing field) | Show the error message, fix and retry |
| 5xx | server / nginx issue | Retry once after 5s; if still failing, tell user to check `systemctl status memos` |

## Idempotency & safety

- **No idempotency keys in v0.29.1** — if a POST times out, do NOT retry blindly; GET recent memos first to check if it landed, then decide.
- **PATCH is partial** — only include fields you want to change; omitted fields keep their current value.
- **DELETE is permanent** — always confirm with the user first; show the memo content before deleting.

## What NOT to do

- Don't write to `demo.canmanciu.top` via this skill — that's SSH-only, no API.
- Don't touch `sale.goibuy.cn` at all.
- Don't log or echo the token in plaintext to the user. If you must reference it, say `$MEMOS_TOKEN`.
- Don't auto-publish without explicit confirmation. PRIVATE is the safe default.
- Don't create memos that contain other people's PII without redaction.
- Don't use this skill as a general-purpose scratchpad — if the user just wants to remember something within this conversation, use a TODO or a code comment, not a memo.

## Reference

- Server topology (verified 2026-10-29): Memos runs as systemd `memos.service` on `root@1.12.55.32` (CentOS 9, Tencent VM) — binary `/usr/local/bin/memos`, SQLite at `/var/lib/memos/memos_prod.db`, listens on 127.0.0.1:5230 behind nginx. This skill only uses the HTTPS API; server ops (upgrade/backup) go via SSH (key `~/.ssh/id_ed25519` works from the Windows PC).
- Upgrade runbook (done 2026-10-29, 0.29.1 → 0.31.0): backup with `sqlite3 memos_prod.db "VACUUM INTO ..."` + copy binary, download release tarball + verify `checksums.txt`, `systemctl stop memos && install -m 0755 memos /usr/local/bin/memos && systemctl start memos`, migrations apply automatically. Backups kept in `/root/backup-memos-upgrade/`.

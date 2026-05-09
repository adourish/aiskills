# Skill: Amplenote API — Notes, Bodies & Token Refresh

**Category:** API Integration
**Complexity:** Intermediate
**Last Updated:** 2026-05-08

## Quick Reference
**Use when:** Creating/reading/updating Amplenote notes via API, refreshing an expired token, migrating content  
**Don't use when:** Simple Amplenote UI tasks — do those in the app  
**Trigger phrases:** "create an amplenote note", "update amplenote", "amplenote token expired", "write to amplenote", "refresh amplenote token"  
**Time:** Token refresh ~2 min; note creation instant  
**Credentials:** `G:\My Drive\03_Areas\Keys\Environments\api-amplenote.json`

---

## Critical API Facts

| What | Endpoint | Notes |
|------|----------|---------|
| Create note (metadata only) | `POST /v4/notes` | Body text in POST is **ignored** — only name + tags saved |
| Set/replace note body | `PUT /v4/notes/{uuid}` | Returns 400 bad_request in practice — **use INSERT_NODES instead** |
| Insert structured nodes | `POST /v4/notes/{uuid}/actions` | Use `INSERT_NODES` for all body content (text, tasks, headings, bullets) |
| Read full note + body | `GET /v4/notes/{uuid}` | Returns body in `body` field (not `text`) |
| List notes | `GET /v4/notes` | Returns metadata only — no body content |

**Always add tags.** Notes without tags are hard to find. Tags go in the POST body as `[{"text": "tagname"}]`.

---

## Correct Flow: Create a Note With Body Content

### Step 1 — Load credentials
```python
import json, urllib.request, urllib.parse, time

config = json.load(open('/tmp/amplenote-config.json'))
token = config['credentials']['accessToken']
```

### Step 2 — POST to create (metadata + tags only)
```python
note_data = {
    'name': 'My Note Title',
    'tags': [{'text': 'tag1'}, {'text': 'tag2'}]
    # DO NOT put body here — it is silently ignored
}
body = json.dumps(note_data).encode()
req = urllib.request.Request(
    'https://api.amplenote.com/v4/notes',
    data=body,
    headers={'Authorization': f'Bearer {token}', 'Content-Type': 'application/json'}
)
with urllib.request.urlopen(req, timeout=20) as r:
    result = json.loads(r.read())
    uuid = result['uuid']
```

### Step 3 — INSERT_NODES to write the body
```python
lines = ["# My Heading", "", "Body text goes here.", "Second paragraph."]
nodes = [
    {'type': 'paragraph', 'content': [{'type': 'text', 'text': l if l.strip() else ' '}]}
    for l in lines
]
# Insert in batches of 15
for i in range(0, len(nodes), 15):
    batch = nodes[i:i+15]
    action = json.dumps({'type': 'INSERT_NODES', 'nodes': batch}).encode()
    req = urllib.request.Request(
        f'https://api.amplenote.com/v4/notes/{uuid}/actions',
        data=action,
        headers={'Authorization': f'Bearer {token}', 'Content-Type': 'application/json'}
    )
    with urllib.request.urlopen(req, timeout=20) as r:
        pass  # 204 = success
    time.sleep(0.3)
```

> **Note:** `PUT /v4/notes/{uuid}` returns 400 bad_request — do not use it. INSERT_NODES is the only reliable way to write body content.

---

## Reading a Note Body

```python
req = urllib.request.Request(
    f'https://api.amplenote.com/v4/notes/{uuid}',
    headers={'Authorization': f'Bearer {token}'}
)
with urllib.request.urlopen(req, timeout=15) as r:
    data = json.loads(r.read())
    body = data.get('body', '')   # body field, NOT text
```

---

## INSERT_NODES — Rich Content (Tasks, Headings, Bullets)

```python
nodes = [
    {'type': 'heading', 'attrs': {'level': 2}, 'content': [{'type': 'text', 'text': 'My Section'}]},
    {'type': 'paragraph', 'content': [{'type': 'text', 'text': 'Some text here.'}]},
    {'type': 'check_list_item', 'attrs': {}, 'content': [
        {'type': 'paragraph', 'content': [{'type': 'text', 'text': 'A task item'}]}
    ]},
    {'type': 'bullet_list_item', 'content': [
        {'type': 'paragraph', 'content': [{'type': 'text', 'text': 'A bullet'}]}
    ]},
]
action = json.dumps({'type': 'INSERT_NODES', 'nodes': nodes}).encode()
req = urllib.request.Request(
    f'https://api.amplenote.com/v4/notes/{uuid}/actions',
    data=action,
    headers={'Authorization': f'Bearer {token}', 'Content-Type': 'application/json'}
)
with urllib.request.urlopen(req, timeout=20) as r:
    pass  # 204 = success
```

Insert in batches of 15 nodes max to avoid API limits. Sleep 0.3s between batches.

---

## Token Refresh — Full OAuth Flow

Tokens expire every 2 hours. Refresh tokens also expire (invalid_grant = full re-auth needed).

### Option A — Refresh token still valid
```python
data = urllib.parse.urlencode({
    'grant_type': 'refresh_token',
    'refresh_token': config['credentials']['refreshToken'],
    'client_id': config['oauth']['clientId'],
}).encode()
req = urllib.request.Request(
    'https://api.amplenote.com/oauth/token',
    data=data,
    headers={'Content-Type': 'application/x-www-form-urlencoded'}
)
with urllib.request.urlopen(req, timeout=15) as r:
    result = json.loads(r.read())
    config['credentials']['accessToken'] = result['access_token']
    if 'refresh_token' in result:
        config['credentials']['refreshToken'] = result['refresh_token']
    json.dump(config, open('/tmp/amplenote-config.json', 'w'), indent=2)
```

### Option B — Both tokens expired (invalid_grant error)

1. **Generate auth URL and give to user:**
```python
params = urllib.parse.urlencode({
    'client_id': config['oauth']['clientId'],
    'redirect_uri': config['oauth']['redirectUri'],   # http://localhost:8080/callback
    'response_type': 'code',
    'state': 'claude_session'
})
print(f"https://login.amplenote.com/login?{params}")
```

2. **User opens URL, authorizes, browser fails on localhost:8080 — that's expected**

3. **User copies the full URL from address bar** (contains `?code=XXXX`)

4. **Exchange code for tokens:**
```python
code = 'paste_code_from_url_here'
data = urllib.parse.urlencode({
    'grant_type': 'authorization_code',
    'code': code,
    'redirect_uri': config['oauth']['redirectUri'],
    'client_id': config['oauth']['clientId'],
}).encode()
req = urllib.request.Request(
    'https://api.amplenote.com/oauth/token',
    data=data,
    headers={'Content-Type': 'application/x-www-form-urlencoded'}
)
with urllib.request.urlopen(req, timeout=15) as r:
    result = json.loads(r.read())
    config['credentials']['accessToken'] = result['access_token']
    config['credentials']['refreshToken'] = result['refresh_token']
    json.dump(config, open('/tmp/amplenote-config.json', 'w'), indent=2)
```

5. **Save fresh tokens back to Google Drive** (`api-amplenote.json`, file ID: `1tEsF0kBL_wZm-Z_7BoZL1y2vsicaojiX`)

---

## OAuth Config Reference

| Field | Value |
|-------|-------|
| clientId | `b889d2968aaee9169fc6981dcf175c2f63af8cddf1bfdce0a431fa1757534502` |
| redirectUri | `http://localhost:8080/callback` |
| authUrl | `https://login.amplenote.com/login` |
| tokenUrl | `https://api.amplenote.com/oauth/token` |
| Credentials file (Drive) | `api-amplenote.json` (ID: `1tEsF0kBL_wZm-Z_7BoZL1y2vsicaojiX`) |

---

## Tag Best Practices

- Always tag notes — untagged notes are impossible to find via API (`GET /v4/notes?tag=X`)
- Tags are sent as `[{"text": "tagname"}]` in the POST body
- Use existing tag taxonomy: `daily-plan`, `taekwondo`, `forms`, `inbox`, `reference`, etc.
- The `GET /v4/notes?tag=tagname` filter uses the tag text value

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Access token expired | Refresh token (Option A above) |
| invalid_grant | Refresh token also expired | Full re-auth (Option B above) |
| Note created but no body | Used `text` in POST body | Use INSERT_NODES after POST |
| Body field empty on GET | Used wrong field name | Use `data.get('body', '')` not `text` |
| PUT returns 400 | API doesn't support PUT body writes | Use INSERT_NODES instead |
| INSERT_NODES fails | Batch too large | Split into batches of ≤15 nodes |

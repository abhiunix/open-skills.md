# Implement Slack File Upload Feature (with optional message)

## Objective
When I mention a file in chat (e.g., "upload somefileName.txt to #my-automation slack channel"), implement the upload using Slack's new file upload API. There is no existing upload script — you must create a reusable `slack_upload.py` script that acts as an importable library AND can be run standalone via CLI.

If I also include a message (e.g., "upload cert-fix.sh with message 'latest fix for the SSL issue'"), attach that message when sharing the file to the channel.

---

## Pre-requisites
Before writing or running any code:

1. **Check `.env` file** — Verify that both `SLACK_TOKEN` and `SLACK_CHANNEL` exist in the project's `.env` file.
2. **If either is missing** — Stop immediately. Ask me to add the missing values before proceeding. Do not assume or fabricate tokens.
3. **Default channel** — If `SLACK_CHANNEL` is not set in `.env` and I haven't specified a channel in my prompt, default to: `my-automation`

---

## Script Design: `slack_upload.py`

Create this as a **reusable module** matching the project's primary tech stack (default: Python).

### Requirements
- **Importable** — Other scripts can do:
```python
  from slack_upload import upload_file, send_message

  upload_file("path/to/file.sh", channel="#my-automation", message="Here's the fix")
  send_message("Deployment complete!", channel="#my-automation")
```
- **CLI-compatible** — Can also be run directly:
```bash
  python slack_upload.py <file_path> [--channel <name_or_id>] [--message "optional message"]
```
- Uses `python-dotenv` to load `.env`
- Uses `requests` for all HTTP calls
- Uses `argparse` for CLI argument parsing

### Exposed Functions
| Function | Purpose |
|---|---|
| `upload_file(file_path, channel=None, message=None)` | Upload a file with optional message |
| `send_message(text, channel=None)` | Send a plain text message to a channel |
| `get_channel_id(token, channel_name)` | Resolve a channel name to its ID |

---

## Slack File Upload API Flow (3-Step Process)

### Step 1: Get an Upload URL
**Endpoint:** `POST https://slack.com/api/files.getUploadURLExternal`

**Headers:**
```
Authorization: Bearer <SLACK_TOKEN>
Content-Type: application/x-www-form-urlencoded
```

**Parameters:**
- `filename` — Name of the file (e.g., `cert-fix.sh`)
- `length` — File size in bytes

**Response:**
```json
{
  "ok": true,
  "upload_url": "https://files.slack.com/upload/v1/...",
  "file_id": "F0XXXXXXXX"
}
```

### Step 2: Upload the File Content
**Endpoint:** `POST <upload_url>` (from Step 1 response)

**Body:** Multipart form-data with the file binary under the key `file`

**Expected:** HTTP 200 on success

### Step 3: Complete the Upload & Share to Channel
**Endpoint:** `POST https://slack.com/api/files.completeUploadExternal`

**Headers:**
```
Authorization: Bearer <SLACK_TOKEN>
Content-Type: application/x-www-form-urlencoded
```

**Parameters:**
- `files` — JSON string: `[{"id": "<file_id>", "title": "<filename>"}]`
- `channel_id` — The Slack channel ID to share the file in
- `initial_comment` *(optional)* — A text message to post alongside the file

**Response:**
```json
{
  "ok": true,
  "files": [...]
}
```

---

## Sending a Message (without file)
If I ask to just send a message (no file), use:

**Endpoint:** `POST https://slack.com/api/chat.postMessage`

**Headers:**
```
Authorization: Bearer <SLACK_TOKEN>
Content-Type: application/json
```

**Body:**
```json
{
  "channel": "<channel_id>",
  "text": "<message>"
}
```

---

## Channel Resolution
If a channel name is provided instead of an ID, resolve it:

**Endpoint:** `GET https://slack.com/api/conversations.list`
**Headers:** `Authorization: Bearer <SLACK_TOKEN>`
**Params:** `types=public_channel,private_channel`

Loop through the `channels` array, match by `name` field, return the `id`. If no match is found, surface a clear error: `"Channel '<name>' not found."`

---

## Environment
- `SLACK_TOKEN` and `SLACK_CHANNEL` live in the project `.env` file
- Load with `python-dotenv`
- Channel priority: **explicit prompt mention > .env value > `my-automation` fallback**

---

## Implementation Rules
1. Resolve the file I mention to its full path in the project directory tree
2. Confirm the file exists before starting upload — fail fast with a clear error if not
3. Use Python with `requests` — no Slack SDK dependency
4. Print clear emoji-prefixed status at each step: preparing → uploading → finalizing → done/error
5. Handle errors at every API step and surface the exact Slack API error message
6. Never hardcode tokens — always read from `.env`
7. If I include a message in my prompt, pass it as `initial_comment` in Step 3
8. The script must work both as an import and as a CLI tool

---

## Example Prompts I Will Use
- "I want to implement slack upload feature for somefile.txt when the script ends"
- "send check_cert_validity.sh to #my-automation"
- "upload temp.txt to #security-team with message 'updated temp config'"
- "send a message 'deployment done' to #my-automation"
- "upload the cert cleanup script with message 'v2 cleanup for Netskope certs'"

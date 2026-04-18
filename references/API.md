# Videolink REST API Reference

Base URL: `https://api.govideolink.com/v1`

Authentication: `Authorization: Bearer <access_token>` or `x-api-key: <api_key>`

---

## Video Endpoints

### GET `/search/videos`

**Search videos by semantic similarity**

Search videos by natural language query using semantic vector search.
Returns matching videos with their AI-generated summary, transcript,
and the most relevant timestamped moments.

Results include deep links with timestamp parameters that open the
video player at the exact moment.

**Parameters:**

| Name | In | Type | Required | Description |
|------|----|------|----------|-------------|
| `q` | query | string | Yes | Natural language search query |
| `limit` | query | number | No | Maximum number of results (1-20, default 5) |

**Response** (200): `VideoSearchApiResponse`

| Field | Type | Required |
|-------|------|----------|
| `results` | VideoSearchResult[] | Yes |
| `query` | string | Yes |

---
### GET `/videos`

**Gets all videos**

Retrieves a list of all existing videos.

**Parameters:**

| Name | In | Type | Required | Description |
|------|----|------|----------|-------------|
| `start` | query | string | No | Provide video id to start the listing after |
| `limit` | query | string | No | Provide a number of videos per page (default: 10) |
| `includeAnalysisData` | query | boolean | No | Set to `true` to inline the full AI-generated `analysis` object on
every returned video. Each inlined `analysis` contains the video's
transcript (full text + timed segments) and its summary (summary
text + structured notes), for the primary transcript language.

Defaults to `false`. For a list endpoint this matters: a page of 10
analyzed videos with inline data can be hundreds of KB to several MB
of transcript payload. Keep the default for grid / browse views.
Only opt in when you actually need the transcripts (e.g. an agent
batch-processing multiple videos). `analysisStatus` is populated
per video regardless, so callers can decide per-item whether to
fetch the full payload via `GET /videos/{id}/analysis`. |

**Response** (200): `VideosListModel`

| Field | Type | Required |
|-------|------|----------|
| `docs` | VideoReadModel[] | Yes |
| `limit` | number | Yes |
| `prev` | string | No |
| `next` | string | No |
| `user` | string | No |

---
### POST `/videos`

**Create a video and get an upload URL**

Creates a new video and returns a short-lived URL to upload the
video bytes to.

Three-step upload flow:

  1. `POST /v1/videos` (this endpoint) — describes the video. The
     response includes a short-lived `uploadUrl` and an `uploadId`.
  2. `PUT <uploadUrl>` with the video bytes and the headers from
     `uploadHeaders` (always include `Content-Type`).
  3. `POST /v1/videos/{id}/finalize` with the `uploadId` to confirm
     the upload and start optimization + transcription.

The API never sees the video bytes; they are streamed directly from
your client to the storage backend over the URL returned here.

**Request body** (`application/json`, VideoCreateRequest):

| Field | Type | Required |
|-------|------|----------|
| `name` | string | Yes |
| `fileType` | string | Yes |
| `autoAnalysis` | boolean | No |

**Response** (201): `VideoCreateResponse`

| Field | Type | Required |
|-------|------|----------|
| `id` | string | Yes |
| `uploadUrl` | string | Yes |
| `uploadMethod` | string | Yes |
| `uploadHeaders` | Record_string.string_ | No |
| `uploadFields` | Record_string.string_ | No |
| `uploadId` | string | Yes |
| `expiresAt` | number | Yes |
| `storagePath` | string | Yes |
| `capability` | object | Yes |

---
### GET `/videos/{id}`

**Finds videos by Id**

Retrieves the details of an existing video.
Supply the unique video ID and receive corresponding video details.

**Parameters:**

| Name | In | Type | Required | Description |
|------|----|------|----------|-------------|
| `id` | path | string | Yes | video Id |
| `includeAnalysisData` | query | boolean | No | Set to `true` to inline the full AI-generated `analysis` object on the
response. The `analysis` payload contains the video's transcript
(full text + timed segments) and its summary (summary text + structured
notes), for the primary transcript language. Use this when you want
the transcript and summary data alongside the video metadata in a
single round-trip.

Defaults to `false` to keep responses small. When omitted or `false`,
only `properties.analysisStatus` is populated (a lightweight flag
object telling you whether transcript + summary exist) so the caller
can decide whether to fetch the full payload via
`GET /videos/{id}/analysis`. |

**Response** (200): `VideoReadModel`

| Field | Type | Required |
|-------|------|----------|
| `id` | string | Yes |
| `userId` | string | Yes |
| `organizationId` | string | Yes |
| `folderId` | string, nullable | No |
| `participants` | Contact[] | No |
| `participantsEmails` | string[] | No |
| `editors` | ContactWithId[] | No |
| `invitationMessage` | string | No |
| `userName` | string | No |
| `createdAt` | string | Yes |
| `editedAt` | string | No |
| `state` | MeetingStatus | Yes |
| `topicId` | string, nullable | Yes |
| `topicName` | string | No |
| `meetingId` | string, nullable | No |
| `meetingName` | string | No |
| `meetingStartsAt` | string | No |
| `meetingEndsAt` | string | No |
| `meetingParentId` | string | No |
| `type` | BlockType | Yes |
| `parentId` | string | No |
| `childrenIds` | string[] | No |
| `videoPageId` | string, nullable | Yes |
| `videoRequestId` | string, nullable | Yes |
| `createdBy` | CreatedBy | No |
| `properties` | object | Yes |
| `analysis` | VideoAnalysisData | No |

**Errors:** 422 — 'id' is required

---
### DELETE `/videos/{id}`

**Deletes an video by id**

Deletes an existing video. Supply the unique video ID to delete the corresponding video.

**Parameters:**

| Name | In | Type | Required | Description |
|------|----|------|----------|-------------|
| `id` | path | string | Yes | Video Id |

**Response** (200): `RepositoryWriteSuccess`

| Field | Type | Required |
|-------|------|----------|
| `writeTime` | object | Yes |

---
### POST `/videos/{id}/finalize`

**Finalize a video upload**

Confirms an upload and starts post-processing.

Call this after `PUT`-ing the video bytes to the URL returned by
`POST /v1/videos`. The endpoint:

  - verifies the uploaded file is present and its size matches
    (within 5% to allow for encoding differences)
  - records the video's storage location
  - starts background optimization and, when enabled on the
    organization, transcription and summary generation

The response is the full video object so you can read the final
URL, preview, and analysis status in a single round trip.
Idempotent: calling finalize twice with the same `uploadId` returns
the same response.

Requires editor access to the video.

**Parameters:**

| Name | In | Type | Required | Description |
|------|----|------|----------|-------------|
| `id` | path | string | Yes | Video ID returned from `POST /v1/videos` |

**Request body** (`application/json`, VideoFinalizeRequest):

| Field | Type | Required |
|-------|------|----------|
| `uploadId` | string | Yes |
| `checksum` | string | No |

**Response** (200): `VideoReadModel`

| Field | Type | Required |
|-------|------|----------|
| `id` | string | Yes |
| `userId` | string | Yes |
| `organizationId` | string | Yes |
| `folderId` | string, nullable | No |
| `participants` | Contact[] | No |
| `participantsEmails` | string[] | No |
| `editors` | ContactWithId[] | No |
| `invitationMessage` | string | No |
| `userName` | string | No |
| `createdAt` | string | Yes |
| `editedAt` | string | No |
| `state` | MeetingStatus | Yes |
| `topicId` | string, nullable | Yes |
| `topicName` | string | No |
| `meetingId` | string, nullable | No |
| `meetingName` | string | No |
| `meetingStartsAt` | string | No |
| `meetingEndsAt` | string | No |
| `meetingParentId` | string | No |
| `type` | BlockType | Yes |
| `parentId` | string | No |
| `childrenIds` | string[] | No |
| `videoPageId` | string, nullable | Yes |
| `videoRequestId` | string, nullable | Yes |
| `createdBy` | CreatedBy | No |
| `properties` | object | Yes |
| `analysis` | VideoAnalysisData | No |

**Errors:** 403 — Insufficient permissions, 404 — Video not found, 409 — Finalization failed — file not found or size mismatch

---
### GET `/videos/{id}/analysis`

**Get video transcript and summary**

Retrieves the AI-generated transcript and summary for a video.

If the video has already been analyzed, returns the existing transcript
and summary immediately. If not, kicks off the analysis pipeline and
either returns `202 Accepted` (default) or waits for completion when
`waitForAnalysis=true` is passed.

The wait is bounded by `timeoutMs` (default 60s, hard-capped at 120s).
If the HTTP caller disconnects mid-wait, the wait is aborted — the
pipeline itself continues running in the background, so a subsequent
call will see the completed result.

**Parameters:**

| Name | In | Type | Required | Description |
|------|----|------|----------|-------------|
| `id` | path | string | Yes | Video ID |
| `waitForAnalysis` | query | boolean | No | When true, block until analysis finishes (or timeout) |
| `timeoutMs` | query | number | No | Max wait in milliseconds (default 60000, max 120000) |
| `language` | query | string | No | Optional language code (defaults to the primary transcript language) |

**Response** (200): `VideoAnalysisResponse`

| Field | Type | Required |
|-------|------|----------|
| `videoId` | string | Yes |
| `status` | string | Yes |
| `languageCode` | string | No |
| `transcript` | VideoTranscriptView | No |
| `summary` | VideoSummaryView | No |
| `processedAt` | string | No |
| `startedAt` | string | No |
| `timeoutMs` | number | No |
| `message` | string | No |

**Errors:** 404 — Video not found

---
### POST `/videos/{id}/share`

**Share a video**

Share a video with specific users or an organization.
Adds the provided emails as participants with editor access.
Requires the caller to be the video owner or have editor access.

**Parameters:**

| Name | In | Type | Required | Description |
|------|----|------|----------|-------------|
| `id` | path | string | Yes | Video ID |

**Request body** (`application/json`, ShareWriteModel):

| Field | Type | Required |
|-------|------|----------|
| `emails` | string[] | No |
| `shareWithOrganization` | boolean | No |

**Response** (200): `ShareResponse`

| Field | Type | Required |
|-------|------|----------|
| `success` | boolean | Yes |
| `sharedWith` | string[] | Yes |

**Errors:** 403 — Insufficient permissions, 404 — Video not found

---

## OAuth Endpoints

### POST `/mcp/oauth/token`

**Get an access token (authorization code + PKCE exchange, or refresh)**

**Request body** (`application/x-www-form-urlencoded`):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `grant_type` | string | Yes | `"authorization_code"`, `"refresh_token"`, or `"client_credentials"` |
| `code` | string | When grant_type=authorization_code | Authorization code from the OAuth flow |
| `redirect_uri` | string | When grant_type=authorization_code | Must match the original redirect |
| `code_verifier` | string | When grant_type=authorization_code | PKCE code verifier |
| `client_id` | string | Yes (if not in Basic auth header) | From DCR or CIMD |
| `client_secret` | string | Yes for agents (client_credentials + agent refresh) | Issued at DCR. Prefer HTTP Basic auth. |
| `refresh_token` | string | When grant_type=refresh_token | Refresh token to rotate |
| `scope` | string | When grant_type=client_credentials | Requested scope (default: `mcp:read`) |

**Client authentication:**
- **Human MCP clients** are public (no secret) and use authorization_code + PKCE.
- **Agent clients** are confidential: they must present `client_secret` via
  HTTP Basic auth (`Authorization: Basic base64(client_id:client_secret)`) or
  in the form body on every `client_credentials` token request AND every agent
  `refresh_token` rotation. A leaked refresh_token alone is not enough to
  obtain new tokens — the client_secret is also required.

**Response** (200):

| Field | Type | Description |
|-------|------|-------------|
| `access_token` | string | Bearer token for API calls |
| `token_type` | string | Always `"Bearer"` |
| `expires_in` | number | Seconds until expiry |
| `refresh_token` | string | Rotated refresh token |

---

## MCP Endpoint

| Method | Path | Description |
|--------|------|-------------|
| POST | /mcp | MCP Streamable HTTP endpoint (Bearer auth required) |
| GET | /.well-known/oauth-authorization-server | OAuth 2.0 Authorization Server metadata (RFC 8414) |
| GET | /.well-known/openid-configuration | OIDC Discovery (alias for AS metadata) |
| GET | /.well-known/oauth-protected-resource | Protected Resource metadata (RFC 9728) |
| POST | /mcp/oauth/register | Dynamic Client Registration (RFC 7591) |
| GET | /mcp/oauth/authorize | Authorization endpoint (redirects to consent page) |
| POST | /mcp/oauth/authorize | Authorization code issuance (with PKCE) |
| POST | /mcp/oauth/token | Token exchange (authorization_code + PKCE, refresh_token, client_credentials) |
| GET | /mcp/oauth/jwks | JSON Web Key Set (empty, opaque tokens) |

**End users should not need a client id.** Minimal config:

```json
{
  "mcpServers": {
    "videolink": {
      "url": "https://api.govideolink.com/v1/mcp"
    }
  }
}
```

MCP clients discover OAuth endpoints via `/.well-known/oauth-authorization-server`. Clients that support CIMD (VS Code, Claude) use URL-based client IDs; others (Cursor, ChatGPT) use Dynamic Client Registration.

---

## Agent Registration (DCR with agent_metadata)

### POST `/mcp/oauth/register` (agent mode)

Include `agent_metadata` in the DCR request to register as an AI agent:

**Request body** (`application/json`):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `client_name` | string | Yes | Agent display name |
| `redirect_uris` | string[] | No | Not required for agents (only needed for authorization code flow) |
| `agent_metadata` | object | Yes (for agents) | Marks this client as an agent |
| `agent_metadata.agent_name` | string | No | Human-readable agent name (defaults to client_name) |
| `agent_metadata.agent_role` | string | No | What the agent does (e.g. "Records demo videos for PRs") |
| `agent_metadata.agent_model` | string | No | LLM powering the agent (e.g. "claude-sonnet-4-6") |
| `agent_metadata.agent_platform` | string | No | Platform/tool (e.g. "Claude Code", "Cursor") |
| `agent_metadata.on_behalf_of` | string | No | Human the agent works for (email or name) |

Agent clients are confidential — they MUST use `token_endpoint_auth_method=client_secret_basic`
(or omit the field; it defaults to `client_secret_basic` for agents).

**Response** (201):

| Field | Type | Description |
|-------|------|-------------|
| `client_id` | string | Public client identifier |
| `client_secret` | string | High-entropy secret, **returned exactly once**. Store immediately. |
| `client_secret_expires_at` | number | 0 = never expires (rotate by re-registering) |
| `token_endpoint_auth_method` | string | Always `client_secret_basic` for agents |
| `agent_claim_code` | string | Human-readable code (e.g. `AGT-X7K9M2`) for org claiming |
| `grant_types` | string[] | Includes `client_credentials` for agents |

Agents start in a placeholder organization. To access any organization's videos,
the `agent_claim_code` must be redeemed by an existing org member via the
web app (Settings > Agents).

---

Full interactive Swagger UI: https://api.govideolink.com/v1/docs/

Swagger JSON: https://api.govideolink.com/v1/swagger.json

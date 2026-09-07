```markdown
# FastServer HTTP API Specification

HTTP endpoints for controlling servers, managing files, by FastServerAPI.

**Base URL**: `http://localhost:4001/api`  
**Listen Port**: `4001`

## Contents

- [Overview](#overview)
- [Health](#health)
- [API Root](#root)
- [Server Management](#servers)
- [Console](#console)
- [Files](#files)
- [Jobs](#jobs)
- [Errors](#errors)

## Overview

All endpoints live under /api unless noted. CORS is enabled. Only GET, POST, and OPTIONS are allowed.

| Key | Value |
|-----|-------|
| CORS | Access-Control-Allow-Origin: *, Methods: GET, POST, OPTIONS, Headers: content-type, authorization, x-fs-key |
| Content Types | application/json (most endpoints), application/octet-stream for file upload/download |
| Notes | Timestamps are Unix seconds. Some OK responses add { timestamp } automatically. |

#### Enable API & Authentication

API is disabled unless enabled in a global config. If a key is present, you must include a Bearer token or x-fs-key header.

```
# Enable the API server and set a password from FastServer settings
```

#### Auth Headers

Use either Authorization: Bearer <key> or x-fs-key: <key>.

```
curl -H 'Authorization: Bearer $FS_API_KEY' \
  'http://localhost:4001/api/servers'

# or
curl -H 'x-fs-key: $FS_API_KEY' \
  'http://localhost:4001/api/servers'
```

#### Common Limits

Payload size limits and safety limits enforced by the server.

```
JSON body: up to 512 KB
Command body: up to 8 KB
Text read: up to 1 MB
Upload (binary): up to 64 MB
Fetch (remote download): up to 100 MB
Unzip: up to 200 MB total, max 5000 entries
```

## Health

Liveness endpoints.

### GET /check_alive

Process-level health check (outside /api).

**Auth**: `May require auth if key is set`

#### Responses

**Status**: `200`

```
{
  "status": "ok",
  "message": "server is running",
  "timestamp": 1738888888,
  "uptime": "0 hours 10 minutes",
  "version": "1.0.0"
}
```

### GET /api/check_alive

API-level health check.

**Auth**: `May require auth if key is set`

#### Responses

**Status**: `200`

```
{
  "status": "ok",
  "uptime": "0 hours 10 minutes",
  "timestamp": 1738888888
}
```

### GET /api/check_plan

Return current plan (pro or normal)

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "ok": true,
  "plan": "pro"
}
```

**Status**: `200`

```
{
  "ok": true,
  "plan": "normal"
}
```

#### Examples

```
curl -s -H "Authorization: Bearer $FS_API_KEY" \
  'http://localhost:4001/api/check_plan' | jq
```

## API Root

### GET /api

API root probe.

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "ok": true,
  "message": "fastserver api root",
  "timestamp": 1738888888
}
```

**Status**: `401`

When API key is configured and header missing/invalid

```
{
  "error": "Unauthorized"
}
```

**Status**: `403`

When api: false in global config

```
{
  "error": "API disabled"
}
```

## Server Management

List servers, inspect details, and control lifecycle.

### GET /api/servers

List registered servers discovered via global config.

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "servers": [
    {
      "id": "srv-1",
      "name": "Survival",
      "software": "Paper",
      "version": "1.20.4",
      "path": "/path/to/server",
      "status": "RUNNING"
    }
  ],
  "timestamp": 1738888888
}
```

#### Examples

```
curl -s -H "Authorization: Bearer $FS_API_KEY" \
  'http://localhost:4001/api/servers' | jq
```

### GET /api/servers/{id}

Get server metadata merged from global config and info.fss.

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "id": "srv-1",
  "servername": "Survival",
  "software": "Paper",
  "version": "1.20.4",
  "serverpath": "/path/to/server",
  "status": "RUNNING",
  "exists": true,
  "timestamp": 1738888888
}
```

**Status**: `404`

```
{
  "error": "Server not found: srv-unknown"
}
```

### GET /api/servers/{id}/status

Get reconciled runtime status (RUNNING/STOPPED/STARTING/STOPPING).

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "id": "srv-1",
  "status": "RUNNING",
  "timestamp": 1738888888
}
```

### POST /api/servers/{id}/start

Start a server (requires Server.jar). Uses configured Java and memory bounds.

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "ok": true,
  "status": "STARTING",
  "timestamp": 1738888888
}
```

**Status**: `500`

```
{
  "status": "false",
  "message": "`Server.jar` not found",
  "timestamp": 1738888888,
  "uptime": "…"
}
```

#### Examples

```
curl -X POST -H "Authorization: Bearer $FS_API_KEY" \
  'http://localhost:4001/api/servers/srv-1/start'
```

### POST /api/servers/{id}/stop

Stop a running server.

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "ok": true,
  "status": "STOPPING",
  "timestamp": 1738888888
}
```

**Status**: `200`

When already stopped

```
{
  "ok": true,
  "status": "NOT_RUNNING",
  "timestamp": 1738888888
}
```

### POST /api/servers/{id}/restart

Restart the server (stop then start).

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "ok": true,
  "status": "STARTING",
  "timestamp": 1738888888
}
```

### POST /api/servers/{id}/command

Send a console command to a running server owned by the API.

**Auth**: `Required if key is set`

**Request Body**

JSON body — Max 8 KB

```
{
  "command": "string (required)"
}
```

#### Responses

**Status**: `200`

```
{
  "ok": true,
  "command": "say hello",
  "timestamp": 1738888888
}
```

**Status**: `500`

server not running

```
{
  "status": "false",
  "message": "Server is not running",
  "timestamp": 1738888888,
  "uptime": "…"
}
```

**Status**: `500`

externally owned

```
{
  "status": "false",
  "message": "This server process is externally owned; cannot send command",
  "timestamp": 1738888888,
  "uptime": "…"
}
```

#### Examples

```
curl -X POST -H "Authorization: Bearer $FS_API_KEY" \
  -H 'Content-Type: application/json' \
  --data '{"command":"say hello"}' \
  'http://localhost:4001/api/servers/srv-1/command'
```

## Console

### GET /api/servers/{id}/console/tail

Tail logs/latest.log. Lines default 200, max 2000.

**Auth**: `Required if key is set`

**Query Parameters**

- `lines` (int, optional, default: 200) — Number of tail lines (1..2000)

#### Responses

**Status**: `200`

```
{
  "id": "srv-1",
  "lines": 200,
  "log": [
    "[12:00:00] Server starting...",
    "..."
  ],
  "timestamp": 1738888888
}
```

**Status**: `500`

missing path or file

```
{
  "status": "false",
  "message": "latest.log not found",
  "timestamp": 1738888888,
  "uptime": "…"
}
```

#### Examples

```
curl -s -H "Authorization: Bearer $FS_API_KEY" \
  'http://localhost:4001/api/servers/srv-1/console/tail?lines=200' | jq
```

## Files

All paths are relative to the server base directory. Dangerous paths (absolute, traversal, wildcards) are rejected.

### GET /api/servers/{id}/files/list

List a directory.

**Auth**: `Required if key is set`

**Query Parameters**

- `path` (string, optional, default: ) — Relative directory (empty = base)

#### Responses

**Status**: `200`

```
{
  "path": "",
  "items": [
    {
      "name": "logs",
      "type": "directory",
      "size": 0
    },
    {
      "name": "server.properties",
      "type": "file",
      "size": 1234
    }
  ],
  "timestamp": 1738888888
}
```

#### Examples

```
curl -s -H "Authorization: Bearer $FS_API_KEY" \
  'http://localhost:4001/api/servers/srv-1/files/list?path=logs' | jq
```

### GET /api/servers/{id}/files/read

Read a small text file (<= 1 MB).

**Auth**: `Required if key is set`

**Query Parameters**

- `path` (string, required) — Relative file path

#### Responses

**Status**: `200`

```
{
  "path": "server.properties",
  "content": "server-port=25565\n...",
  "timestamp": 1738888888
}
```

**Status**: `500`

```
{
  "status": "false",
  "message": "File too large to read (>1MB)",
  "timestamp": 1738888888,
  "uptime": "…"
}
```

### POST /api/servers/{id}/files/write

Write text file (creates or overwrites).

**Auth**: `Required if key is set`

**Request Body**

JSON body — Max 512 KB

```
{
  "path": "string (required)",
  "content": "string (required)"
}
```

#### Responses

**Status**: `200`

```
{
  "path": "server.properties",
  "ok": true,
  "timestamp": 1738888888
}
```

#### Examples

```
curl -X POST -H "Authorization: Bearer $FS_API_KEY" \
  -H 'Content-Type: application/json' \
  --data '{"path":"server.properties","content":"server-port=25565\n"}' \
  'http://localhost:4001/api/servers/srv-1/files/write'
```

### POST /api/servers/{id}/files/create

Create an empty file.

**Auth**: `Required if key is set`

**Request Body**

JSON body — Max 512 KB

```
{
  "path": "string (required)"
}
```

#### Responses

**Status**: `200`

```
{
  "path": "README.txt",
  "ok": true,
  "timestamp": 1738888888
}
```

### POST /api/servers/{id}/files/mkdir

Create a directory.

**Auth**: `Required if key is set`

**Request Body**

JSON body — Max 512 KB

```
{
  "path": "string (required)",
  "parents": "boolean (default true)"
}
```

#### Responses

**Status**: `200`

```
{
  "path": "conf/newdir",
  "ok": true,
  "timestamp": 1738888888
}
```

### POST /api/servers/{id}/files/delete

Delete a file or directory. Non-recursive dirs must be empty.

**Auth**: `Required if key is set`

**Request Body**

JSON body — Max 512 KB

```
{
  "path": "string (required)",
  "recursive": "boolean (default false)"
}
```

#### Responses

**Status**: `200`

```
{
  "deleted": "old.log",
  "ok": true,
  "timestamp": 1738888888
}
```

**Status**: `500`

dir not empty (recursive=false)

```
{
  "status": "false",
  "message": "Directory not empty",
  "timestamp": 1738888888,
  "uptime": "…"
}
```

### POST /api/servers/{id}/files/rename

Move/rename file or directory.

**Auth**: `Required if key is set`

**Request Body**

JSON body — Max 512 KB

```
{
  "from": "string (required)",
  "to": "string (required)",
  "overwrite": "boolean (default false)"
}
```

#### Responses

**Status**: `200`

```
{
  "from": "a.txt",
  "to": "b.txt",
  "ok": true,
  "timestamp": 1738888888
}
```

### POST /api/servers/{id}/files/copy

Copy file or directory.

**Auth**: `Required if key is set`

**Request Body**

JSON body — Max 512 KB

```
{
  "from": "string (required)",
  "to": "string (required)",
  "overwrite": "boolean (default false)"
}
```

#### Responses

**Status**: `200`

```
{
  "from": "a.txt",
  "to": "copy/a.txt",
  "ok": true,
  "timestamp": 1738888888
}
```

### POST /api/servers/{id}/files/upload

Upload binary file.

**Auth**: `Required if key is set`

**Query Parameters**

- `path` (string, required) — Target relative path
- `overwrite` (boolean, optional, default: true) — Allow overwrite

**Request Body**

Binary body — Max 64 MB

#### Responses

**Status**: `200`

```
{
  "path": "plugins/my.jar",
  "bytes": 12345,
  "ok": true,
  "timestamp": 1738888888
}
```

**Status**: `500`

payload too large

```
{
  "status": "false",
  "message": "Payload too large",
  "timestamp": 1738888888,
  "uptime": "…"
}
```

#### Examples

```
curl -X POST -H "Authorization: Bearer $FS_API_KEY" \
  --data-binary @plugin.jar \
  'http://localhost:4001/api/servers/srv-1/files/upload?path=plugins/plugin.jar&overwrite=true'
```

### GET /api/servers/{id}/files/download

Download a file as attachment. Returns binary, not JSON.

**Auth**: `Required if key is set`

**Query Parameters**

- `path` (string, required) — Relative file path

#### Responses

**Status**: `200`

Content-Type probed; Content-Disposition set

**Status**: `500`

```
{
  "status": "false",
  "message": "File not found or invalid",
  "timestamp": 1738888888,
  "uptime": "…"
}
```

#### Examples

```
curl -OJ -H "Authorization: Bearer $FS_API_KEY" \
  'http://localhost:4001/api/servers/srv-1/files/download?path=server.properties'
```

### POST /api/servers/{id}/files/fetch

Server downloads from a remote http/https URL and saves.

**Auth**: `Required if key is set`

**Request Body**

JSON body — Max 512 KB

```
{
  "url": "string (http/https required)",
  "path": "string",
  "overwrite": "boolean (default true)"
}
```

#### Responses

**Status**: `200`

```
{
  "path": "downloads/file.bin",
  "bytes": 9999,
  "ok": true,
  "timestamp": 1738888888
}
```

**Status**: `500`

remote too large / bad protocol

```
{
  "status": "false",
  "message": "Only http/https is allowed",
  "timestamp": 1738888888,
  "uptime": "…"
}
```

### POST /api/servers/{id}/files/zip

Create a ZIP archive from a directory (async job).

**Auth**: `Required if key is set`

**Request Body**

JSON body — Max 512 KB

```
{
  "source": "string (directory)",
  "zip_path": "string (zip file path)"
}
```

#### Responses

**Status**: `200`

```
{
  "job": {
    "id": "jobid",
    "type": "zip",
    "status": "queued"
  },
  "timestamp": 1738888888
}
```

### POST /api/servers/{id}/files/unzip

Extract a ZIP archive to a directory (async job). Safe extraction enforced.

**Auth**: `Required if key is set`

**Request Body**

JSON body — Max 512 KB

```
{
  "zip_path": "string (zip file)",
  "dest": "string (target directory)"
}
```

#### Responses

**Status**: `200`

```
{
  "job": {
    "id": "jobid",
    "type": "unzip",
    "status": "queued"
  },
  "timestamp": 1738888888
}
```

## Jobs

Track progress and cancel async ZIP/UNZIP jobs.

### GET /api/jobs

List all jobs.

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "jobs": [
    {
      "id": "abc123",
      "type": "zip",
      "status": "running",
      "progress": 55,
      "processed_bytes": 1234567,
      "total_bytes": 4560000,
      "source": "",
      "target": "",
      "started_at": 1738888800
    }
  ],
  "timestamp": 1738888888
}
```

### GET /api/jobs/{id}

Inspect a job.

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "id": "abc123",
  "type": "unzip",
  "status": "completed",
  "progress": 100,
  "processed_bytes": 15000,
  "total_bytes": 15000,
  "source": "downloads/archive.zip",
  "target": "world/",
  "started_at": 1738888800,
  "finished_at": 1738888860,
  "message": "Unzip completed",
  "timestamp": 1738888888
}
```

**Status**: `404`

```
{
  "error": "Job not found"
}
```

### POST /api/jobs/{id}/cancel

Attempt to cancel a running job.

**Auth**: `Required if key is set`

#### Responses

**Status**: `200`

```
{
  "id": "abc123",
  "type": "zip",
  "status": "canceled",
  "message": "Job canceled",
  "finished_at": 1738888890,
  "timestamp": 1738888890
}
```

**Status**: `500`

```
{
  "status": "false",
  "message": "Failed to cancel job",
  "timestamp": 1738888888,
  "uptime": "…"
}
```

#### Examples

```
curl -X POST -H "Authorization: Bearer $FS_API_KEY" \
  'http://localhost:4001/api/jobs/abc123/cancel' | jq
```

## Errors

Error payload shape varies: some endpoints use { error }, others use { status: "false", message }.

#### Path Safety

Absolute paths, backslashes, colons, traversal (..), and wildcards (*, ?, [ ]) are rejected. All operations are constrained within the server base directory.

### — Common status codes

#### Responses

**Status**: `401`

```
{
  "error": "Unauthorized"
}
```

**Status**: `403`

```
{
  "error": "API disabled"
}
```

**Status**: `404`

```
{
  "error": "Unknown API path: ..."
}
```

**Status**: `405`

```
{
  "error": "Method Not Allowed"
}
```

**Status**: `500`

```
{
  "status": "false",
  "message": "Error message",
  "timestamp": 1738888888,
  "uptime": "…"
}
```
```
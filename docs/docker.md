# Running easy-notion-mcp with Docker

## Quick Start

Pull the latest image from GitHub Container Registry and run with an API token:

```bash
docker pull ghcr.io/grey-iris/easy-notion-mcp:latest
docker run -p 3333:3333 \
  -e NOTION_TOKEN=ntn_your_token_here \
  ghcr.io/grey-iris/easy-notion-mcp:latest
```

Access the server at `http://localhost:3333/mcp`

## Build Locally

```bash
docker build -t notion-mcp:latest .
docker run -p 3333:3333 \
  -e NOTION_TOKEN=ntn_your_token_here \
  notion-mcp:latest
```

## HTTP Server Modes

### Static Token (API Token)

```bash
docker run -p 3333:3333 \
  -e NOTION_TOKEN=ntn_your_token_here \
  ghcr.io/grey-iris/easy-notion-mcp:latest
```

### OAuth

```bash
docker run -p 3333:3333 \
  -e NOTION_OAUTH_CLIENT_ID=your_client_id \
  -e NOTION_OAUTH_CLIENT_SECRET=your_client_secret \
  -e OAUTH_REDIRECT_URI=http://localhost:3333/callback \
  ghcr.io/grey-iris/easy-notion-mcp:latest
```

Visit `http://localhost:3333` to authorize.

## Stdio Mode

For stdio transport (e.g., MCP clients that use stdin/stdout):

```bash
docker run -i \
  -e NOTION_TOKEN=ntn_your_token_here \
  ghcr.io/grey-iris/easy-notion-mcp:latest \
  node dist/index.js
```

## Using Environment Files

Create a `.env` file with your configuration:

```bash
docker run -p 3333:3333 \
  --env-file .env \
  ghcr.io/grey-iris/easy-notion-mcp:latest
```

## Environment Variables

### Required (choose one auth method)

| Variable                                               | Mode                   | Description                           |
|--------------------------------------------------------|------------------------|---------------------------------------|
| `NOTION_TOKEN`                                         | HTTP (static) or Stdio | Notion internal integration token     |
| `NOTION_OAUTH_CLIENT_ID`, `NOTION_OAUTH_CLIENT_SECRET` | HTTP (OAuth)           | OAuth credentials for user-level auth |

### Optional

| Variable               | Default                            | Description                            |
|------------------------|------------------------------------|----------------------------------------|
| `PORT`                 | 3333                               | HTTP server port                       |
| `NOTION_ROOT_PAGE_ID`  | ~                                  | Default parent page for creating pages |
| `NOTION_TRUST_CONTENT` | ~                                  | Skip content notice prefix             |
| `OAUTH_REDIRECT_URI`   | `http://localhost:{PORT}/callback` | OAuth callback URL                     |

# nable-rmm-mcp

**N-able RMM** MCP server — exposes the N-able RMM Data Extraction API as MCP tools.

> **Naming note:** this integration is called **"N-able RMM"** in app.mspbots.ai (`sys_integration.subject_code = SOLARWIND` — it was originally SolarWinds RMM). N-able's own public developer docs call the same product **"N-sight"** (`https://developer.n-able.com/n-sight/docs/...`). This is a different product from **N-able N-Central**, which is a separate integration in app.mspbots.ai with its own API. This server wraps N-sight/N-able RMM only.

<div align="center">

## MSPbots Connector

**The MCP Gateway built for MSPs: visible, controlled, and your data stays where it belongs.**

[**Get started at connect.mspbots.ai →**](https://connect.mspbots.ai)

</div>

---

MSPbots Connector gives your AI clients one MCP URL to reach your entire MSP stack (RMM, PSA, documentation, security, finance), and gives you control over who can call which tool and a record of everything that was called.

**Hosted service: [connect.mspbots.ai](https://connect.mspbots.ai)** · Free to use.

### Why Connector

![MSPbots Connector](https://stgoss.mspbots.ai/files/mcp/mspbots-mcp.png)

| | |
|---|---|
| **Visible** | Every tool call is logged: who, which vendor, when. Usage dashboard, CSV export, SIEM streaming. |
| **Controlled** | Role-based, per-tool allowlist. Deny by default. |
| **Protected** | Credentials encrypted per record. AI clients never see your vendor passwords or tokens. |
| **MSP-native** | Built for the MSP stack only: RMM, PSA, documentation, security, finance. |

### Quick start

1. **Sign up** at [connect.mspbots.ai](https://connect.mspbots.ai).
2. **Connect your vendors.** Pick from 150+ connectors and authorize each one on the vendor's own page, or bring your own API keys. No extra vendor accounts needed.
3. **Set access.** Create roles and decide which tools each role may call. Nothing is allowed until you allow it.
4. **Add your MCP URL to your AI client.** Copy the URL from your Connector dashboard into Claude, ChatGPT, Cursor, VS Code, Gemini, Codex, or any MCP-capable client.
5. **Ask your AI to do real work**, and watch every call appear in the audit log.

### Features

**Unified gateway**
- One MCP URL for all your AI clients
- 150+ connectors across the MSP stack
- Custom MCP servers supported, free
- Authorization happens on each vendor's own page; the AI never sees passwords or tokens
- Bring your own credentials: use the vendor API keys you already have
- Multiple accounts per app; connect once, reuse across AI clients
- Tool search: finds the right action for the task
- One sign-in; vendor credentials are injected behind the scenes and OAuth tokens refresh automatically

**Visibility**
- Audit log for every tool call (who, which vendor, when)
- Usage dashboard: call volume, top vendors, error rate, per-user activity
- Export audit logs to CSV, or stream to Loki, Graylog, or LogScale (SIEM)
- Optional recording of tool arguments and result summaries (off by default, enabled per organization)

**Control**
- Per-role, per-tool allowlist, deny by default
- owner / admin / member roles
- SCIM provisioning with Entra ID, Okta, JumpCloud, and Google Workspace

**Security**
- Credentials encrypted one record at a time (AES-256-GCM, per-record key)
- AI clients authenticate with OAuth 2.1 + PKCE
- Vendor responses are scanned for prompt-injection patterns and returned as structured data. This is one layer of defense, not a guarantee against every attack.
- Static egress IP for customer firewall and API allowlisting

### Pricing

**Free.**

Links: [connect.mspbots.ai](https://connect.mspbots.ai) · [mspbots.ai](https://mspbots.ai)

---

## Overview

This server implements the [Model Context Protocol](https://modelcontextprotocol.io/) (Streamable HTTP/SSE transport) and wraps the [N-sight API](https://developer.n-able.com/n-sight/docs/getting-started-with-the-n-sight-api) (the API behind the "N-able RMM" integration). It follows the MSPbots **Vendor MCP Service SOP**: stateless, no stored credentials, per-request header authentication.

The underlying API itself authenticates via an `apikey` query parameter and returns XML. This server never stores that key — the caller passes it per-request via a header, and every tool response is the XML converted to JSON for readability.

## Quick Start

### Docker (recommended)

```bash
docker compose up --build
```

The server starts on `http://localhost:8080`.

### Local (uv)

```bash
uv sync
python -m nable_rmm_mcp
```

## Health Check

```bash
curl http://localhost:8080/health
# {"status": "ok"}
```

No token is required for the health endpoint.

## Authentication

Every request to `/mcp` must include the following HTTP headers. Header naming mirrors the fields on the existing "N-able RMM" integration form in app.mspbots.ai (**API Token** / **URL**).

| Header | Type | Required | Default | Allowed values | Description | Example |
|---|---|---|---|---|---|---|
| `X-Nable-Api-Token` | string | Required | None | None (free text) | N-able RMM API key, generated at **Settings > General Settings > API > Generate**. If the key is restricted to a client group, calls only return data for clients in that group. | `X-Nable-Api-Token: <your_api_key>` |
| `X-Nable-Server` | string | Optional | `NABLE_BASE_URL` environment variable (default `https://www.am.remote.management`) | See the list of 11 regional servers below | N-able RMM is split into 11 independent servers by the region an account belongs to. Without this header the environment variable default is used; when the gateway serves tenants from multiple regions at once, it is overridden per request. | `X-Nable-Server: https://www.systemmonitor.co.uk` |

The 11 allowed values of `X-Nable-Server` (region -> Base URL):

| Region | Base URL |
|---|---|
| Americas (default) | `https://www.am.remote.management` |
| Asia | `https://wwwasia.system-monitor.com` |
| Australia | `https://www.system-monitor.com` |
| Europe | `https://wwweurope1.systemmonitor.eu.com` |
| France | `https://wwwfrance.systemmonitor.eu.com` |
| France1 | `https://wwwfrance1.systemmonitor.eu.com` |
| Germany | `https://wwwgermany1.systemmonitor.eu.com` |
| Ireland | `https://wwwireland.systemmonitor.eu.com` |
| Poland | `https://wwwpoland1.systemmonitor.eu.com` |
| United Kingdom | `https://www.systemmonitor.co.uk` |
| United States | `https://www.systemmonitor.us` |

Missing `X-Nable-Api-Token` returns `401 Unauthorized`. N-able RMM also caps automated calls at **60 requests/minute per key** — this server does not throttle for you, so pace bulk calls accordingly.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `MCP_HTTP_PORT` | `8080` | Listening port |
| `MCP_HTTP_HOST` | `0.0.0.0` | Listening host |
| `NABLE_BASE_URL` | `https://www.am.remote.management` | Default regional server; override per-request with `X-Nable-Server` |

## MCP Endpoint

```
POST http://localhost:8080/mcp
```

Connect your MCP client with:
- Transport: `http` (Streamable HTTP / SSE)
- Headers: `X-Nable-Api-Token: <api_key>` (required), `X-Nable-Server: <base_url>` (optional)

## Tool List

16 tools, covering the full public N-sight API reference plus the 4 endpoints already in production use under app.mspbots.ai's "N-able RMM" integration (Device Monitoring Details, Drive Space History, Patch, Performance History).

### Clients & Sites

| Tool | Description | Parameters |
|---|---|---|
| `nable_rmm_get_clients` | List all clients visible to this API key | `devicetype?` ("server"/"workstation"/"mobile_device") |
| `nable_rmm_get_sites` | List all sites under a client | `clientid` (required, int) |
| `nable_rmm_create_client` | Create a new client (write operation) | `name` (required), `timezone?`, `officehoursemail?`, `officehourssms?`, `outofofficehoursemail?`, `outofofficehourssms?` |
| `nable_rmm_create_site` | Create a new site under a client (write operation) | `clientid` (required), `sitename` (required), `router1?`, `router2?`, `workstationtemplate?`, `servertemplate?` |

### Devices

| Tool | Description | Parameters |
|---|---|---|
| `nable_rmm_get_devices_at_client` | List devices across all sites of a client by device type | `clientid` (required), `devicetype` (required, "server"/"workstation"/"mobile_device") |
| `nable_rmm_get_servers` | List server devices under a site with online/check status | `siteid` (required, int) |
| `nable_rmm_get_workstations` | List workstation devices under a site with online/check status | `siteid` (required, int) |
| `nable_rmm_get_agentless_assets` | List agentless (scan-discovered) assets under a site (N-able has marked this feature as deprecated) | `siteid` (required, int) |
| `nable_rmm_get_device_monitoring_details` | Get detailed monitoring information for a single device | `deviceid` (required, int) |

### Checks, Patch & Performance

| Tool | Description | Parameters |
|---|---|---|
| `nable_rmm_get_failing_checks` | List currently failing checks | `clientid?`, `check_type?` ("checks"/"tasks"/"random") |
| `nable_rmm_get_checks` | List all checks configured on a device | `deviceid` (required, int) |
| `nable_rmm_get_check_config` | Get the detailed configuration of a single check (structure varies by check type/OS) | `checkid` (required, int) |
| `nable_rmm_get_outages` | List outage records for a device (open, or closed within the last 61 days) | `deviceid` (required, int) |
| `nable_rmm_get_drive_space_history` | Disk space check history (filtered from list_checks by check_type 1003/1004; N-sight has no dedicated service) | `deviceid` (required, int) |
| `nable_rmm_get_patch_list` | List all patches on a device with status/severity | `deviceid` (required, int) |
| `nable_rmm_get_performance_history` | Historical performance metrics for a device (bandwidth/disk/CPU/memory/NIC) | `deviceid` (required, int), `interval` (required, int, must be 15 or 60), `since?` |

## Test Example

List all servers at a site:

```json
{
  "method": "tools/call",
  "params": {
    "name": "nable_rmm_get_servers",
    "arguments": { "siteid": 12345 }
  }
}
```

Equivalent `curl` against the running server (streamable HTTP MCP endpoint):

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "X-Nable-Api-Token: <api_key>" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "nable_rmm_get_servers",
      "arguments": { "siteid": 12345 }
    }
  }'
```

Expected shape of a successful result (the underlying XML response, converted to JSON; fields abbreviated):

```json
{
  "content": [
    {
      "type": "text",
      "text": "{\n  \"created\": \"2026-03-15T10:00:00+00:00\",\n  \"host\": \"...\",\n  \"status\": \"OK\",\n  \"items\": {\n    \"server\": [ { \"serverid\": \"9001\", \"name\": \"DC01\", \"online\": \"1\", \"status_247\": \"5\", \"os\": \"Windows Server 2022\" } ]\n  }\n}"
    }
  ]
}
```

## API Reference

- [N-sight API — Getting Started](https://developer.n-able.com/n-sight/docs/getting-started-with-the-n-sight-api)
- [Generate an API Key](https://developer.n-able.com/n-sight/docs/generate-an-api-key)
- [Determine the server for your API query](https://developer.n-able.com/n-sight/docs/determine-server-for-api-query)

## Verified Against a Live Account

All 14 read tools have been exercised end-to-end against a real N-able RMM account (US region) and returned real data, including edge cases N-able's public docs left ambiguous:
- `nable_rmm_get_performance_history`'s `interval` parameter is **required** (not optional as the docs example implied) and only accepts `15` or `60` — confirmed via the API's own validation error (`Invalid "interval": must be 15 or 60 (minutes)`).
- `nable_rmm_get_agentless_assets` returns empty `items` on this account, consistent with N-able's note that the feature is deprecated.

The two write tools (`nable_rmm_create_client`, `nable_rmm_create_site`) were **not** exercised against the live account to avoid creating real data in a production customer's N-able RMM tenant — verify these against a disposable test account before relying on them.

## Known Gaps / Not Implemented

- `get_site_installation_package` — returns a binary installer download rather than structured data, not practical as an MCP tool.

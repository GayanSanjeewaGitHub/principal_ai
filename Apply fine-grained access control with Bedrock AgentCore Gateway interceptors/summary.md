# Fine-Grained Access Control with Amazon Bedrock AgentCore Gateway Interceptors

## Overview

As enterprises scale AI adoption to hundreds of agents and thousands of MCP tools, managing secure, dynamic access becomes a critical challenge. **AgentCore Gateway interceptors** address this through fine-grained access control, schema translation, and flexible authorization — all without creating operational bottlenecks or per-tenant gateway instances.

---

## Core Problems Addressed

| Challenge | Description |
|---|---|
| **Tool access governance** | Filtering MCP tool access per calling principal (agent, user, app) based on dynamic permissions |
| **Schema translation & data protection** | Mapping MCP schemas to downstream APIs while redacting PII/SPI before it reaches backend services |
| **Tenant isolation** | Enforcing multi-tenant boundaries without deploying separate gateways per tenant |
| **Dynamic tool filtering** | Real-time, no-cache permission filtering on `ListTools`, `InvokeTool`, and `Search` calls |
| **Identity propagation** | Safely passing user identity across multi-hop workflows without leaking over-privileged tokens |

---

## Gateway Interceptors

Two interception points are available in the request-response lifecycle:

### 1. Request Interceptor (Lambda)
Runs **before** the request reaches the target tool. Use cases:
- JWT validation and scope-based access control
- Blocking unauthorized tool invocations
- Injecting scoped-down authorization headers
- Schema/payload transformation

**Input payload structure:**
```json
{
  "interceptorInputVersion": "1.0",
  "mcp": {
    "gatewayRequest": {
      "headers": { "Authorization": "Bearer eyJhbG..." },
      "body": { "jsonrpc": "2.0", "method": "tools/list" }
    },
    "requestContext": { ... }
  }
}
```

**Required output:**
```json
{
  "interceptorOutputVersion": "1.0",
  "mcp": {
    "transformedGatewayRequest": { ... }
  }
}
```

### 2. Response Interceptor (Lambda)
Runs **before** the response returns to the calling agent. Use cases:
- Filtering tool lists to only authorized tools
- Audit trail creation
- Schema translation on response payloads

**Required output:**
```json
{
  "interceptorOutputVersion": "1.0",
  "mcp": {
    "transformedGatewayResponse": { ... }
  }
}
```

---

## Access Control: JWT Scope-Based Authorization

Scopes follow a predictable naming convention:
- Full target access: `mcp-target-123`
- Tool-level access: `mcp-target-123:getOrder`

### Core authorization check (Python):
```python
def check_tool_authorization(scopes, tool, target):
    if target in scopes:
        return True
    return f"{target}:{tool}" in scopes
```

### Request interceptor flow:
1. Extract and decode JWT → retrieve `scope` claim
2. Identify invoked tool (`tools/call`)
3. Block if user lacks full target or tool-specific permission
4. Return structured MCP error for unauthorized calls

---

## Dynamic Tool Filtering (Response Interceptor)

Tools returned from `tools/list` or semantic search are filtered before reaching the agent:

```python
def lambda_handler(event, context):
    gateway_response = event['mcp']['gatewayResponse']
    auth_header = gateway_response['headers'].get('Authorization', '')
    token = auth_header.replace('Bearer ', '')
    claims = decode_jwt_payload(token)
    scopes = claims.get('scope', '').split()

    tools = gateway_response['body']['result'].get('tools', [])
    if not tools:
        tools = gateway_response['body']['result'] \
            .get('structuredContent', {}).get('tools', [])

    filtered_tools = filter_tools_by_scope(tools, scopes)
    return {
        "interceptorOutputVersion": "1.0",
        "mcp": {
            "transformedGatewayResponse": {
                "statusCode": 200,
                "headers": {"Authorization": auth_header},
                "body": {"result": {"tools": filtered_tools}}
            }
        }
    }

def filter_tools_by_scope(tools, allowed_scopes):
    filtered_tools = []
    for tool in tools:
        target, action = tool['name'].split('___')
        if target in allowed_scopes or f"{target}:{action}" in allowed_scopes:
            filtered_tools.append(tool)
    return filtered_tools
```

---

## Identity Propagation: Impersonation vs. Act-on-Behalf

| Approach | How it works | Recommendation |
|---|---|---|
| **Impersonation** | Original JWT passed unchanged through every hop | ❌ Not recommended — risk of privilege escalation, confused deputy attacks |
| **Act-on-behalf** | Each hop receives a separate, scoped token for that specific downstream target | ✅ Recommended — least privilege, limited blast radius, full auditability |

**Act-on-behalf benefits:**
- Principle of least privilege per downstream service
- Clear audit chain via AgentCore Observability
- Prevents confused deputy attacks
- Scoped tokens can't be reused across services

### Custom header propagation (Request Interceptor):
```python
def lambda_handler(event, context):
    gateway_request = event.get('mcp', {}).get('gatewayRequest', {})
    headers = gateway_request.get('headers', {})
    body = gateway_request.get('body', {})
    auth_header = headers.get('authorization', '') or headers.get('Authorization', '')

    if "arguments" in body["params"]:
        body["params"]["arguments"]["authorization"] = auth_header

    return {
        "interceptorOutputVersion": "1.0",
        "mcp": {
            "transformedGatewayRequest": {
                "headers": {
                    "Accept": "application/json",
                    "Authorization": auth_header,
                    "Content-Type": "application/json"
                },
                "body": body
            }
        }
    }
```

---

## No Auth + OAuth Hybrid Model

Enterprises can configure **No Auth** at the gateway level for open tool discovery, while enforcing OAuth at method level via interceptors:

- `ListTools` / `SearchTools` → No authentication required (open discovery)
- `CallTools` → JWT validation enforced by request interceptor

**Gateway creation with No Auth:**
```json
{
  "name": "no-auth-gateway",
  "protocolType": "MCP",
  "protocolConfiguration": {
    "mcp": { "supportedVersions": ["2025-03-26"] }
  },
  "authorizerType": "NONE",
  "roleArn": "<role-arn>"
}
```

---

## Observability (AgentCore + CloudWatch)

Interceptors automatically integrate with **AgentCore Observability** and emit metrics/logs to CloudWatch:

| Feature | Detail |
|---|---|
| **Security decision visibility** | Allow/deny decisions with evaluated JWT scopes |
| **Request/response traceability** | Logs header enrichment, schema translation, data redaction |
| **Downstream tool observability** | Status codes, latency, error responses per target |
| **Multi-tenant context** | Identity attributes to isolate issues per tenant/user group |

Sample performance: **100% success rate**, **~4.47ms average latency**, no throttling.

---

## Key Design Principles

- Keep interceptors **deterministic** — avoid complex branching or LLM-driven logic
- Rely on **token claims only** (not external state) for authorization decisions
- Never pass original JWT unchanged to downstream APIs
- Use **no caching** for dynamically filtered tool lists — permissions change at any time
- Enforce authorization at the **gateway boundary**, before the LLM sees or executes tools

---

## Deep Dive: Interceptor Execution Model (Synchronous)

The gateway request interceptor is a **fully synchronous, blocking call**. AgentCore Gateway pauses the original request, waits for the interceptor Lambda to respond, and only then decides whether to forward or reject the request. The tool never executes until the interceptor explicitly allows it.

```
MCP Client ──► AgentCore Gateway ──► [PAUSE] ──► Interceptor Lambda
                                                        │
                                     ◄── allow/deny ───┘
                                          │
                              ──► Target Tool (only if allowed)
```

### What the Interceptor Checks

The gateway passes the full raw request to the Lambda, including the `Authorization: Bearer <JWT>` header. The interceptor:

1. **Decodes the JWT** → reads the `scope` claim (e.g. `mcp-target-123 mcp-target-123:getOrder`)
2. **Identifies the tool** from `body.method` (`tools/call`) and the tool name in params
3. **Runs the authorization check:**
```python
def check_tool_authorization(scopes, tool, target):
    if target in scopes:               # full target access
        return True
    return f"{target}:{tool}" in scopes  # tool-level access
```
4. Returns either:
   - ✅ `transformedGatewayRequest` → gateway forwards to the tool
   - ❌ MCP error → gateway rejects, tool never executes

### Important: The Interceptor Does NOT Create Tokens for the Gateway

The scoped JWT is created **upstream** by your identity provider (e.g. Amazon Cognito) before the request hits the gateway. The interceptor only **reads and validates** those scopes. What it *can* do is **replace** the incoming over-privileged JWT with a new scoped-down token targeted at the specific downstream API (act-on-behalf pattern).

### Complete Synchronous Flow

```
1. User/Agent gets JWT from Cognito (with scopes)
2. Calls AgentCore Gateway with JWT
3. Gateway STOPS → invokes Request Interceptor Lambda (sync, waits)
4. Interceptor decodes JWT, checks scopes vs. requested tool
5a. Authorized  → returns transformedGatewayRequest (optionally with new scoped token)
5b. Unauthorized → returns MCP error, gateway rejects, tool never runs
6. Gateway forwards to tool (only if 5a)
```

---

## Deep Dive: Are Scoped-Down Tokens One-Time Use?

**No — they are short-lived, not one-time use by default.** This is an important distinction.

### Token Properties

| Property | Value |
|---|---|
| **Audience (`aud`)** | Scoped to that specific downstream API only |
| **Scope** | Only the permissions needed for that one tool call |
| **Expiry (`exp`)** | Short TTL — typically seconds to a few minutes |
| **Subject (`sub`)** | Still carries the original user identity (act-on-behalf) |
| **One-time use?** | ❌ No, unless explicitly built |

### Why Short-Lived ≠ One-Time Use

A JWT is stateless — the downstream API validates only signature + expiry. There is no built-in mechanism to invalidate a token after first use. The same token can be replayed within its TTL window.

### Options to Achieve True One-Time Use

```
Option 1: JTI (JWT ID) claim + revocation store
  - Interceptor adds a unique `jti` to each generated token
  - Downstream API checks jti against a used-token store (Redis/DynamoDB)
  - Rejects if jti was seen before
  - Con: requires shared state, adds latency

Option 2: Very short TTL (e.g. 30 seconds)
  - Not truly one-time, but limits replay window practically
  - Simplest approach — no shared state needed
  - Most commonly used

Option 3: Token binding
  - Bind token to the TLS session or client certificate
  - Strongest guarantee, hardest to implement
```

### What AgentCore Gateway Uses in Practice

The act-on-behalf pattern uses **short-lived scoped tokens**. The security guarantee comes from three layers:
1. **Narrow scope** — a stolen token can only call that one specific tool
2. **Short TTL** — token expires quickly, limiting replay window
3. **Audience restriction** — token is rejected by any other downstream API

True one-time use requires implementing the `jti` + revocation store yourself inside your interceptor Lambda.

---

## Reference

- [AgentCore Gateway Docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html)
- [Sample Notebook – Fine-Grained Access Control](https://github.com/awslabs/amazon-bedrock-agentcore-samples/blob/main/01-tutorials/02-AgentCore-gateway/09-fine-grained-access-control/01-fine-grained-access-control-using-custom-scopes.ipynb)

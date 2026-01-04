# Clawdbot Security Review

**Date:** 2026-01-04
**Reviewer:** Claude Opus 4.5 (Automated Security Audit)
**Scope:** Full codebase security analysis

---

## Executive Summary

The clawdbot codebase demonstrates **good security practices** in several key areas, with some recommendations for hardening. Overall, the architecture shows security-conscious design with proper credential handling, input validation via Zod schemas, timing-safe authentication, and optional Docker sandbox isolation.

**Risk Level:** LOW to MODERATE
**Known Vulnerabilities in Dependencies:** 0 (as of audit date)

---

## Findings by Category

### 1. Authentication & Authorization

#### Strengths ✅

| Finding | Location | Details |
|---------|----------|---------|
| **Timing-safe password comparison** | `src/gateway/auth.ts:30-33` | Uses `crypto.timingSafeEqual()` with length check to prevent timing attacks |
| **PKCE OAuth flow** | `src/commands/antigravity-oauth.ts` | Implements Proof Key for Code Exchange for OAuth security |
| **Loopback detection** | `src/gateway/auth.ts:35-42` | Properly validates localhost/loopback addresses including IPv6 |
| **Tailscale header validation** | `src/gateway/auth.ts:64-99` | Validates Tailscale proxy headers only from loopback IPs |
| **Forwarded header awareness** | `src/gateway/auth.ts:55-61` | Rejects requests with X-Forwarded-* headers when expecting local direct |

#### Recommendations

| Priority | Issue | Recommendation |
|----------|-------|----------------|
| LOW | Token comparison in `auth.ts:141` uses `===` | Consider using timing-safe comparison for token auth as well (currently password uses it) |
| INFO | Multiple auth extraction methods | Query parameter tokens may appear in logs; prefer header-based auth |

---

### 2. Credential Storage & Secrets Management

#### Strengths ✅

| Finding | Location | Details |
|---------|----------|---------|
| **Restrictive file permissions** | `src/agents/model-auth.ts:61-67` | OAuth directory created with mode `0o700`, files with `0o600` |
| **Credential backup mechanism** | `src/web/session.ts:60-86` | WhatsApp creds backed up before save to prevent corruption |
| **No hardcoded production secrets** | Throughout | Tokens sourced from env vars, files, or config |
| **Legacy credential migration** | `src/agents/model-auth.ts:70-90` | Handles migration from legacy paths securely |

#### Recommendations

| Priority | Issue | Recommendation |
|----------|-------|----------------|
| MEDIUM | Session transcripts in plaintext | `~/.clawdbot/sessions/*.jsonl` stores conversation logs unencrypted - consider at-rest encryption for sensitive deployments |
| LOW | Telegram token in config | `telegram.botToken` in config file is less secure than `TELEGRAM_BOT_TOKEN` env var or `tokenFile` |

---

### 3. Command Injection & Process Execution

#### Strengths ✅

| Finding | Location | Details |
|---------|----------|---------|
| **Docker sandbox isolation** | `src/agents/sandbox.ts` | Comprehensive container isolation with capability dropping, read-only root, tmpfs, network isolation |
| **Process tree killing** | `src/agents/shell-utils.ts:30-52` | Properly kills process groups to prevent orphans |
| **Binary output sanitization** | `src/agents/shell-utils.ts:13-28` | Strips control characters and surrogates from output |
| **Elevated mode gating** | `src/agents/bash-tools.ts:171-174` | Elevated commands require explicit `enabled` AND `allowed` flags |
| **Sandbox path validation** | `src/agents/sandbox-paths.ts` | Validates paths stay within sandbox root, rejects symlinks |

#### Analysis of Sandbox Security (`src/agents/sandbox.ts`)

The Docker sandbox configuration includes strong defaults:
- `--read-only` root filesystem
- `--cap-drop ALL` by default
- `--network none` (no network access)
- `--security-opt no-new-privileges`
- Support for seccomp and AppArmor profiles
- Resource limits (CPU, memory, PIDs, ulimits)
- Configurable tmpfs mounts

#### Recommendations

| Priority | Issue | Recommendation |
|----------|-------|----------------|
| MEDIUM | Sandbox mode defaults to `"off"` | Consider defaulting to `"non-main"` for production deployments |
| LOW | Shell from environment | `src/agents/shell-utils.ts:9` uses `$SHELL` which could be attacker-controlled in some scenarios |
| INFO | Signal-cli spawned with user-provided path | `src/signal/daemon.ts:52` - ensure `cliPath` is validated |

---

### 4. Input Validation & Sanitization

#### Strengths ✅

| Finding | Location | Details |
|---------|----------|---------|
| **Zod schema validation** | `src/config/zod-schema.ts`, `src/config/validation.ts` | All config validated against Zod schema before use |
| **JSON5 parsing with validation** | `src/config/io.ts` | Config file parsed and validated, invalid configs rejected |
| **Hook payload validation** | `src/gateway/hooks.ts:153-228` | Strict type checking and normalization of webhook payloads |
| **Max body size limits** | `src/gateway/hooks.ts:69-109` | 256KB default limit with stream abort on excess |
| **Redirect loop protection** | `src/media/store.ts:46-62` | Max 5 redirects for media downloads |
| **Media size limits** | `src/media/store.ts:11,79-81` | 5MB max with stream abort on excess |

#### Recommendations

| Priority | Issue | Recommendation |
|----------|-------|----------------|
| LOW | No rate limiting | Gateway endpoints lack request rate limiting (potential DoS vector) |
| INFO | Unicode space normalization | `src/agents/sandbox-paths.ts:5-8` handles Unicode spaces - ensure comprehensive coverage |

---

### 5. Network & API Security

#### Strengths ✅

| Finding | Location | Details |
|---------|----------|---------|
| **Method restrictions** | `src/gateway/server-http.ts:93-98` | Hooks endpoint only accepts POST |
| **HTTPS for external requests** | `src/media/store.ts:4` | Media downloads use node:https |
| **Content-Type enforcement** | `src/gateway/server-http.ts:49-52` | Responses include proper Content-Type headers |
| **WebSocket upgrade handling** | `src/gateway/server-http.ts:214,244-256` | Proper upgrade delegation |

#### Recommendations

| Priority | Issue | Recommendation |
|----------|-------|----------------|
| MEDIUM | Error message exposure | `src/gateway/server-http.ts:237-238` returns `String(err)` to client - may leak internal details |
| LOW | Missing security headers | Consider adding `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY` for control UI |
| LOW | HTTP-only media downloads | Consider allowing HTTP in addition to HTTPS for local/private networks |

---

### 6. Dependencies

**Audit Result:** No known vulnerabilities detected (669 dependencies scanned)

Key security-relevant dependencies:
- `@whiskeysockets/baileys@7.0.0-rc.9` - WhatsApp Web protocol (actively maintained)
- `ws@8.18.3` - WebSocket library
- `zod@4.3.4` - Schema validation
- `grammy@1.39.2` - Telegram bot framework

---

### 7. Code Quality & Security Hygiene

#### Strengths ✅

| Finding | Location | Details |
|---------|----------|---------|
| **TypeScript strict mode** | Throughout | Strong typing reduces runtime errors |
| **Biome linting** | `package.json` | Modern linting with security-aware rules |
| **70% coverage threshold** | `package.json:153-154` | Enforced test coverage |
| **Error serialization safety** | `src/web/session.ts:231-259` | Handles circular refs, BigInt, functions |

---

## Risk Assessment Matrix

| Category | Risk Level | Notes |
|----------|------------|-------|
| Authentication | LOW | Timing-safe comparison, PKCE OAuth |
| Credential Storage | LOW | Proper permissions, backup mechanism |
| Command Injection | LOW-MEDIUM | Excellent when sandbox enabled; higher risk with sandbox off + elevated mode |
| Input Validation | LOW | Zod schemas, size limits |
| Network Security | LOW | HTTPS, method restrictions |
| Dependencies | LOW | No known vulnerabilities |

---

## Priority Action Items

### High Priority
*None identified*

### Medium Priority

1. **Enable sandbox by default for production** - Change `sandbox.mode` default from `"off"` to `"non-main"`
2. **Encrypt session transcripts** - Add at-rest encryption for `~/.clawdbot/sessions/*.jsonl`
3. **Sanitize error messages** - Avoid returning raw error strings to clients in HTTP responses

### Low Priority

1. Add rate limiting to gateway endpoints
2. Use timing-safe comparison for token authentication (in addition to password)
3. Add security headers to control UI HTTP responses
4. Validate external CLI paths (signal-cli, imsg) before execution

---

## Secure Configuration Recommendations

```json5
// ~/.clawdbot/clawdbot.json - Hardened configuration example
{
  "agent": {
    "sandbox": {
      "mode": "non-main",  // Enable sandbox for non-main sessions
      "perSession": true,
      "docker": {
        "readOnlyRoot": true,
        "network": "none",
        "capDrop": ["ALL"],
        "pidsLimit": 100,
        "memory": "512m"
      }
    },
    "elevated": {
      "enabled": false,  // Disable elevated mode unless required
      "allowed": false
    }
  },
  "gateway": {
    "auth": {
      "mode": "token"  // Require token authentication
    }
  },
  "hooks": {
    "enabled": true,
    "token": "GENERATE_SECURE_RANDOM_TOKEN",
    "maxBodyBytes": 262144  // 256KB limit
  }
}
```

---

## Files Reviewed

- `src/gateway/auth.ts` - Gateway authentication
- `src/agents/model-auth.ts` - OAuth credential management
- `src/agents/bash-tools.ts` - Bash command execution
- `src/agents/sandbox.ts` - Docker sandbox configuration
- `src/agents/sandbox-paths.ts` - Path validation
- `src/agents/shell-utils.ts` - Shell utilities
- `src/gateway/hooks.ts` - Webhook handling
- `src/gateway/server-http.ts` - HTTP request handling
- `src/media/store.ts` - Media download/storage
- `src/web/session.ts` - WhatsApp session handling
- `src/config/io.ts` - Configuration loading
- `src/signal/daemon.ts` - Signal daemon spawning
- `src/imessage/client.ts` - iMessage RPC client
- `src/telegram/token.ts` - Telegram token handling
- `src/discord/token.ts` - Discord token handling
- `package.json` - Dependencies

---

## Conclusion

The clawdbot codebase demonstrates security-conscious design with no critical vulnerabilities identified. The optional Docker sandbox provides strong isolation when enabled. Primary recommendations focus on enabling sandbox by default for production and encrypting sensitive session data at rest.

For high-security deployments, ensure:
1. Sandbox mode is enabled (`sandbox.mode: "non-main"` or `"all"`)
2. Gateway authentication is enabled (`gateway.auth.mode: "token"`)
3. Elevated mode is disabled unless explicitly required
4. Environment variables are used for sensitive tokens rather than config files

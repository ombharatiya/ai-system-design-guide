# Access Control for LLM Systems

Secure access control is essential for multi-user and multi-tenant LLM applications. This chapter covers authentication, authorization, and data isolation patterns.

## Table of Contents

- [Access Control Requirements](#access-control-requirements)
- [Authentication Patterns](#authentication-patterns)
- [Authorization Models](#authorization-models)
- [Agent and Tool Authorization](#agent-and-tool-authorization)
- [Tenant Isolation](#tenant-isolation)
- [API Key Management](#api-key-management)
- [Audit and Compliance](#audit-and-compliance)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Access Control Requirements

### Security Dimensions

| Dimension | Description | Controls |
|-----------|-------------|----------|
| **Authentication** | Who is making the request? | API keys, OAuth, JWT, workload identity |
| **Authorization** | What can they do? | RBAC, ABAC, policies |
| **Isolation** | What data can they see? | Tenant filtering, encryption |
| **Audit** | What did they do? | Logging, compliance reports |

### LLM-Specific Concerns

| Concern | Risk | Mitigation |
|---------|------|------------|
| Prompt injection | Bypass access controls | Input validation |
| Data leakage | Cross-tenant exposure | Strict filtering |
| Model output | Expose protected info | Output filtering |
| Context pollution | Inject unauthorized data | Context validation |
| Agent credentials | A hijacked agent or assistant session reuses its tokens | Scoped, short-lived, sender-constrained tokens |
| Confused deputy across tool servers | One server steers credentials meant for another | Bind every credential to its issuer and audience |

---

## Authentication Patterns

### API Key Authentication

```python
class APIKeyAuthenticator:
    def __init__(self, key_store):
        self.key_store = key_store
    
    async def authenticate(self, api_key: str) -> AuthResult:
        if not api_key:
            return AuthResult(authenticated=False, error="Missing API key")
        
        # Hash the key for lookup
        key_hash = self.hash_key(api_key)
        
        # Look up in store
        key_record = await self.key_store.get(key_hash)
        
        if not key_record:
            return AuthResult(authenticated=False, error="Invalid API key")
        
        if key_record.expired:
            return AuthResult(authenticated=False, error="Expired API key")
        
        if key_record.revoked:
            return AuthResult(authenticated=False, error="Revoked API key")
        
        return AuthResult(
            authenticated=True,
            user_id=key_record.user_id,
            tenant_id=key_record.tenant_id,
            scopes=key_record.scopes
        )
    
    def hash_key(self, key: str) -> str:
        return hashlib.sha256(key.encode()).hexdigest()
```

### JWT with Scopes

```python
class JWTAuthenticator:
    def __init__(self, public_key: str):
        self.public_key = public_key
    
    async def authenticate(self, token: str) -> AuthResult:
        try:
            payload = jwt.decode(
                token,
                self.public_key,
                algorithms=["RS256"],
                audience="llm-api"
            )
            
            return AuthResult(
                authenticated=True,
                user_id=payload["sub"],
                tenant_id=payload.get("tenant_id"),
                scopes=payload.get("scopes", []),
                expires_at=datetime.fromtimestamp(payload["exp"])
            )
        except jwt.ExpiredSignatureError:
            return AuthResult(authenticated=False, error="Token expired")
        except jwt.InvalidTokenError as e:
            return AuthResult(authenticated=False, error=str(e))
```

### Workload Identity Instead of Static Keys

Static API keys are the weakest link in an agent fleet, and attackers now hunt them by name. The ChainDrop npm worm (August 2026) harvested OpenAI, Anthropic, Cursor, Codex and Gemini tokens alongside cloud and GitHub credentials. Mandiant described an intrusion in which an attacker took over a live AI coding-assistant session, the assistant recommended an already-poisoned package, and stolen GitHub OAuth tokens let the Shai-Hulud worm spread across about 100 internal repositories. Providers have moved accordingly: OpenAI made mutual TLS and X.509 workload identity federation GA (August 29, 2026), added API key expiration with org- and project-level maximum lifetimes (September 10), and lets admins restrict or disable key creation (September 15).

The rules that follow:

- **Prefer workload identity** (mTLS, federated short-lived tokens) for service-to-provider calls. Where keys remain, give each an owner, an expiry and a spend cap.
- **Treat an agent or AI-assistant session as a principal** with its own scoped credentials and monitoring, and keep raw API keys and long-lived OAuth tokens out of its reach.
- **Use sender-constrained tokens** where the protocol supports them (DPoP, RFC 9449), so a stolen bearer token is not enough on its own.

---

## Authorization Models

### Role-Based Access Control (RBAC)

```python
class RBACAuthorizer:
    ROLE_PERMISSIONS = {
        "admin": ["*"],
        "developer": ["generate", "embed", "fine_tune", "read_metrics"],
        "user": ["generate", "embed"],
        "viewer": ["read_metrics"]
    }
    
    def authorize(self, user: User, action: str) -> bool:
        permissions = self.ROLE_PERMISSIONS.get(user.role, [])
        
        if "*" in permissions:
            return True
        
        return action in permissions
```

### Attribute-Based Access Control (ABAC)

```python
class ABACAuthorizer:
    def __init__(self, policy_engine):
        self.policy_engine = policy_engine
    
    async def authorize(
        self,
        subject: dict,       # Who (user attributes)
        action: str,         # What (operation)
        resource: dict,      # On what (resource attributes)
        context: dict        # When/where (environmental)
    ) -> AuthzResult:
        # Evaluate all applicable policies
        policies = await self.policy_engine.get_policies(action)
        
        for policy in policies:
            result = policy.evaluate(subject, action, resource, context)
            if result == PolicyResult.DENY:
                return AuthzResult(allowed=False, reason=policy.name)
            if result == PolicyResult.ALLOW:
                return AuthzResult(allowed=True)
        
        return AuthzResult(allowed=False, reason="No matching policy")
```

### Model-Level Permissions

```python
class ModelAccessControl:
    MODEL_TIERS = {
        "claude-fable-5-1": ["enterprise"],
        "claude-opus-5-5": ["enterprise", "professional"],
        "gpt-6-sol": ["enterprise", "professional"],
        "claude-haiku-4-5": ["enterprise", "professional", "starter"],
        "gpt-6-luna": ["enterprise", "professional", "starter"]
    }
    
    def __init__(self, zdr_eligible: set[str]):
        # Load from your vendor agreements, not from code: retention terms are
        # per model (e.g. Fable 5.1 requires 30-day retention unless authorized)
        self.zdr_eligible = zdr_eligible
    
    def can_access_model(self, user: User, model: str) -> bool:
        if user.tier not in self.MODEL_TIERS.get(model, []):
            return False
        if user.tenant.requires_zdr and model not in self.zdr_eligible:
            return False
        return True
    
    def get_available_models(self, user: User) -> list[str]:
        return [
            model for model in self.MODEL_TIERS
            if self.can_access_model(user, model)
        ]
```

Model access is now gated on the vendor side too. Anthropic's Cyber Verification Program (expanded October 6, 2026, with Project Glasswing folded in) has three tiers (Defense, Red Team, Specialized); Mythos 5.1 is available in every tier, admins assign the program to specific workspaces, and enrollment requires data retention, so the entitlement and the retention term change together. OpenAI's `gpt-rosalind-research` is trusted-access only, and Gemini 3.8 Flash Cyber is available only through Google's Fairwind Program. The model registry behind this check should record, per model and platform, which models and access programs the organization is entitled to, their data-retention terms, and their retirement dates, so routing and fallback can never pick a model a tenant may not use.

---

## Agent and Tool Authorization

When an agent calls tools on a user's behalf, authorization involves three parties: the user, the agent acting as an OAuth client, and each tool server. 2026 produced both a standards direction and a concrete failure.

**The failure: authorization-server mix-up.** The official MCP SDKs let the MCP server decide which authorization server received the client's credentials, so a malicious or compromised server could name its own and receive stored refresh tokens and client secrets with no user interaction (CVE-2026-104850 in TypeScript, GHSA-qx49-fqc8-xw99 in Python, both CVSS 7.5; fixed in TypeScript 1.31.0 and 2.2.0, Python 1.30.0 and 2.2.0). Upgrading alone is not enough: machine-to-machine providers must be configured with the expected issuer (`expectedIssuer` in TypeScript, `issuer=` in Python), and credentials saved before the upgrade stay exposed until they are tagged with their issuer or cleared. A multi-server agent client is a natural confused deputy. The rule: **bind every credential to its issuer, and never let a resource server choose where secrets go.**

**The building blocks:**

| Need | Mechanism | Status |
|------|-----------|--------|
| A token usable only at one server | Audience binding (RFC 8707 resource indicators) | Part of the MCP authorization spec, but the TypeScript SDK accepted tokens issued for other services (CVE-2026-111771); the fix in `sdk` 1.32.0 and `server` 2.3.0 is opt-in, so set `expectedResource` and return `AuthInfo.resource` |
| A stolen token is useless on its own | DPoP sender-constrained tokens (RFC 9449) | Shipped in the MCP TypeScript client 2.1.0 (September 23, 2026); the spec profile (SEP-1932) is still an open draft |
| Least privilege per tool | Per-tool step-up scope challenges (HTTP 403 `insufficient_scope`) | Shipped in the MCP TypeScript server 2.1.0 |
| Enterprise IdP controls which servers an agent may use | ID-JAG grant behind MCP Enterprise-Managed Authorization | Shipped in MCP; the IETF draft is at -04, not yet an RFC |
| The agent's own identity, for headless agents | Workload identity plus the OAuth family, not new protocols | IETF WIMSE adopted draft-ietf-wimse-aims-00 (September 15, 2026); MCP workload identity federation (SEP-1933) is an open draft |

**Approvals are authorization too.** Per-call human approval does not scale, so platforms are moving to policy: Claude Managed Agents' auto permission policy (September 10, 2026) has the server evaluate each agent or MCP tool call and run it, deny it, or pause for approval. Where humans do approve, bind the approval to the exact call: research on approval laundering (arXiv 2609.38983) shows the action a person approves is not always the action a coding-agent harness executes.

**User-delegated model access.** Sign in with ChatGPT (announced at OpenAI DevDay, September 29, 2026; limited trial for selected partners) uses OAuth 2.0 with PKCE and OpenID Connect, and can optionally run eligible Responses API requests on the user's own ChatGPT plan. It is convenient identity, but it concentrates login and AI billing at one model vendor; weigh it like any IdP lock-in decision.

Protocol-level detail, including Enterprise-Managed Authorization, is in [Tool Use and MCP](../07-agentic-systems/03-tool-use-and-mcp.md).

---

## Tenant Isolation

### Data Isolation Patterns

```python
class TenantIsolatedVectorStore:
    def __init__(self, vector_db):
        self.db = vector_db
    
    async def search(
        self,
        tenant_id: str,
        query_embedding: list[float],
        top_k: int = 10
    ) -> list[dict]:
        # CRITICAL: Always filter by tenant_id at database level
        results = await self.db.search(
            query_vector=query_embedding,
            top_k=top_k,
            filter={"tenant_id": {"$eq": tenant_id}}  # Mandatory filter
        )
        
        return results
    
    async def insert(
        self,
        tenant_id: str,
        documents: list[dict]
    ):
        # CRITICAL: Always include tenant_id in metadata
        for doc in documents:
            doc["metadata"]["tenant_id"] = tenant_id
        
        await self.db.insert(documents)
```

### Prompt Isolation

```python
class TenantAwarePromptBuilder:
    def build_prompt(
        self,
        tenant_id: str,
        user_query: str,
        context: list[dict]
    ) -> str:
        # Verify all context belongs to tenant
        for doc in context:
            if doc.get("tenant_id") != tenant_id:
                raise SecurityError("Cross-tenant context detected")
        
        # Build isolated prompt
        return f"""
[Tenant: {tenant_id}]
Context from tenant documents:
{self.format_context(context)}

User query: {user_query}
"""
```

### Cache Isolation

```python
class TenantIsolatedCache:
    def __init__(self, cache_backend):
        self.cache = cache_backend
    
    def _scoped_key(self, tenant_id: str, key: str) -> str:
        return f"tenant:{tenant_id}:{key}"
    
    async def get(self, tenant_id: str, key: str) -> any:
        return await self.cache.get(self._scoped_key(tenant_id, key))
    
    async def set(self, tenant_id: str, key: str, value: any, ttl: int = 3600):
        await self.cache.set(
            self._scoped_key(tenant_id, key),
            value,
            ttl=ttl
        )
```

Cache isolation also reaches into the inference engine. A shared prefix cache is a timing side channel: if tenant B's request returns faster because tenant A already cached the same prefix, B learns something about A's prompt. vLLM isolates tenants with a per-request `cache_salt`, and in September 2026 a regression on one code path (tool-continuation turns on its Harmony `/v1/responses` endpoint dropped the salt) reopened the oracle until 0.30.0 (the current vLLM floor is 0.32.0; see [Securing the AI Stack Itself](01-llm-security.md#securing-the-ai-stack-itself)). Pass a per-tenant salt on every request, test each endpoint for it, and when many tenants share one provider account, keep tenant-specific content after the shared cacheable prefix.

The same failure shows up in protocol servers. In the MCP TypeScript SDK, an HTTP server that created one transport or server instance at startup and shared it across clients could deliver one client's responses to another (CVE-2026-111770, CVSS 7.7, `@modelcontextprotocol/server` 2.0.0 to 2.2.x), because messages were matched to connections only by JSON-RPC id and every SDK client numbers its requests from 0. The SDK's own README examples showed the shared pattern. Version 2.3.0 refuses it, so upgrading breaks those apps until each request or session gets its own server and transport. Any per-connection state in a multi-tenant server, not only caches, needs a tenant or session key.

---

## API Key Management

### Key Lifecycle

```python
class APIKeyManager:
    KEY_PREFIX = "llm_"
    
    async def create_key(
        self,
        user_id: str,
        tenant_id: str,
        name: str,
        scopes: list[str],
        expires_in_days: int = 365
    ) -> APIKey:
        # Generate secure key
        raw_key = self.KEY_PREFIX + secrets.token_urlsafe(32)
        key_hash = self.hash_key(raw_key)
        
        # Store metadata (not the raw key)
        key_record = APIKeyRecord(
            id=generate_id(),
            hash=key_hash,
            user_id=user_id,
            tenant_id=tenant_id,
            name=name,
            scopes=scopes,
            created_at=datetime.now(),
            expires_at=datetime.now() + timedelta(days=expires_in_days)
        )
        
        await self.store.save(key_record)
        
        # Return raw key only once (not stored)
        return APIKey(
            id=key_record.id,
            key=raw_key,  # Only returned on creation
            name=name,
            scopes=scopes,
            expires_at=key_record.expires_at
        )
    
    async def revoke_key(self, key_id: str, reason: str):
        await self.store.update(key_id, {
            "revoked": True,
            "revoked_at": datetime.now(),
            "revoke_reason": reason
        })
        
        await self.audit_log.log("api_key_revoked", {
            "key_id": key_id,
            "reason": reason
        })
```

### Key Rotation

```python
class KeyRotator:
    async def rotate_key(self, old_key_id: str) -> APIKey:
        old_key = await self.key_store.get(old_key_id)
        
        # Create new key with same permissions
        new_key = await self.key_manager.create_key(
            user_id=old_key.user_id,
            tenant_id=old_key.tenant_id,
            name=f"{old_key.name} (rotated)",
            scopes=old_key.scopes
        )
        
        # Grace period: old key still works temporarily
        await self.key_store.update(old_key_id, {
            "deprecated": True,
            "deprecated_at": datetime.now(),
            "grace_period_ends": datetime.now() + timedelta(days=7)
        })
        
        await self.notify_user(old_key.user_id, new_key)
        
        return new_key
```

---

## Audit and Compliance

### Audit Logging

```python
class AuditLogger:
    async def log_request(
        self,
        request: LLMRequest,
        response: LLMResponse,
        auth: AuthResult
    ):
        audit_entry = {
            "timestamp": datetime.now().isoformat(),
            "request_id": request.id,
            "user_id": auth.user_id,
            "tenant_id": auth.tenant_id,
            "action": "llm_generate",
            "model": request.model,
            "input_tokens": response.usage.input_tokens,
            "output_tokens": response.usage.output_tokens,
            "cost": response.cost,
            "latency_ms": response.latency_ms,
            # Hash content for privacy
            "input_hash": self.hash_content(request.prompt),
            "output_hash": self.hash_content(response.content)
        }
        
        await self.audit_store.append(audit_entry)
```

### Compliance Reports

```python
class ComplianceReporter:
    async def generate_report(
        self,
        tenant_id: str,
        start_date: datetime,
        end_date: datetime
    ) -> ComplianceReport:
        logs = await self.audit_store.query(
            tenant_id=tenant_id,
            start=start_date,
            end=end_date
        )
        
        return ComplianceReport(
            tenant_id=tenant_id,
            period=(start_date, end_date),
            total_requests=len(logs),
            unique_users=len(set(l["user_id"] for l in logs)),
            models_used=list(set(l["model"] for l in logs)),
            total_cost=sum(l["cost"] for l in logs),
            data_access_events=self.extract_data_access(logs),
            security_events=await self.get_security_events(tenant_id, start_date, end_date)
        )
```

---

## Interview Questions

### Q: How do you implement multi-tenant isolation in a RAG system?

**Strong answer:**

"Multi-tenant isolation requires defense in depth:

**Vector database level:**
- Every vector includes tenant_id in metadata
- All queries filter by tenant_id at the database level
- Never filter after retrieval (data already leaked to memory)

**Cache level:**
- All cache keys prefixed with tenant_id
- Semantic cache scoped to tenant
- No cross-tenant cache hits even for identical queries

**Prompt level:**
- Validate context documents belong to requesting tenant before including
- Never mix context from multiple tenants

**Output level:**
- Verify response does not contain cross-tenant information
- Output filtering as additional safeguard

**Audit:**
- Log all access with tenant context
- Monitor for cross-tenant access attempts

The key principle: tenant_id is a mandatory filter at every data access point, not an optional parameter."

### Q: How do you manage API keys for an LLM service?

**Strong answer:**

"Secure API key management:

**Creation:**
- Generate cryptographically random keys
- Store only the hash, return raw key once
- Associate with user, tenant, scopes, expiration

**Validation:**
- Hash incoming key, compare to stored hash
- Check expiration and revocation status
- Verify scopes match requested action

**Rotation:**
- Support key rotation with grace period
- Old key works during transition (7 days)
- Notify users of impending expiration

**Security:**
- Rate limit failed authentication attempts
- Revoke immediately on suspected compromise
- Audit all key operations

**Scopes:**
- Fine-grained: model access, operation type, daily limits
- Least privilege by default

**Keys we hold, not just keys we issue:**
- For our own calls to model providers, prefer workload identity (mTLS or federated tokens) over static keys, and give any remaining key an owner, an expiry and a spend cap
- LLM API keys are now standard loot for supply-chain worms, so keep them out of developer machines, agent sessions and CI logs
- A gateway concentrates them: the poisoned LiteLLM releases of October 21, 2026 harvested provider keys and the proxy's master and virtual keys, so I need to be able to rotate everything the gateway holds in one step

The key principle: never store raw keys, support rotation, implement least privilege."

### Q: An agent calls tools on five MCP servers on a user's behalf. How do you authorize it?

**Strong answer:**

"I design against the confused deputy, because the agent holds credentials for five servers and any one of them could be malicious or compromised.

**Bind credentials to issuer and audience.** Each token is audience-bound to its server (RFC 8707), and each stored credential is tagged with the authorization server that issued it and never sent anywhere else. That exact gap was CVE-2026-104850 in the MCP TypeScript SDK in September 2026 (the Python SDK had the same flaw): a malicious server could name its own authorization server and collect refresh tokens and client secrets. Upgrading the SDK is not enough; machine-to-machine clients need an explicit expected issuer, and pre-upgrade credentials must be re-tagged or cleared. On the server side, audience binding only helps if the server checks it: the TypeScript SDK's October fix (CVE-2026-111771) is opt-in, so every server I run sets `expectedResource`.

**Make stolen tokens less useful.** Short lifetimes, and DPoP sender-constrained tokens where the client and server support them.

**Least privilege per tool.** Narrow default scopes with per-tool step-up challenges, so a read-only session cannot silently call a write tool.

**Enterprise control.** Where the company runs an IdP, Enterprise-Managed Authorization lets admins decide centrally which servers the agent may reach. For headless agents, I give the agent its own workload identity rather than a user's long-lived token.

**Approval and audit.** High-risk calls go through a policy engine or a human, with the approval bound to the exact call executed, and every call is logged with user, agent, server and scope.

Standards are still settling (DPoP for MCP and workload identity federation are drafts, ID-JAG is not yet an RFC), so I keep the authorization layer behind my own interface and expect to swap pieces."

---

## References

- OAuth 2.0: https://oauth.net/2/
- OWASP API Security: https://owasp.org/API-Security/
- RFC 8707, Resource Indicators for OAuth 2.0: https://www.rfc-editor.org/rfc/rfc8707
- RFC 9449, OAuth 2.0 Demonstrating Proof of Possession (DPoP): https://www.rfc-editor.org/rfc/rfc9449
- MCP TypeScript SDK advisory GHSA-6qxp-vccf-f47h (CVE-2026-104850): https://github.com/modelcontextprotocol/typescript-sdk/security/advisories/GHSA-6qxp-vccf-f47h
- MCP TypeScript SDK advisories GHSA-jm63-cq47-j5jp (CVE-2026-111770, shared transports) and GHSA-rvq5-wwqv-78pq (CVE-2026-111771, token audience): https://github.com/modelcontextprotocol/typescript-sdk/security/advisories
- Anthropic, expanded Cyber Verification Program (October 6, 2026): https://www.anthropic.com/news/cyber-verification-program
- IETF WIMSE, AI Identity Management System (draft-ietf-wimse-aims): https://datatracker.ietf.org/doc/draft-ietf-wimse-aims/
- OpenAI API changelog (workload identity, key expiry): https://developers.openai.com/api/docs/changelog

---

*Previous: [LLM Security](01-llm-security.md)*

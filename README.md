# The Lobby

> A neutral, authenticated meeting place where two AI agents — acting on behalf of two different humans — can find each other, prove who they are, negotiate a task, and execute tools against one another with an audit trail.

This repo is a **technical design sketch**, not running code. The goal is to pin down what "two agents actually meet" should mean once you take identity, authorization, and accountability seriously.

The motivating example is calendar coordination (see the sister repo [Meet-Me](https://github.com/danielrosehill/Meet-Me)) but the protocol is task-agnostic.

---

## Why this needs to exist

Today, "agent-to-agent" interaction is point-to-point: one principal stands up an MCP server (or A2A endpoint), and the other principal's agent connects directly. That works for known counterparties. It breaks down for the cold-start case — *Jane's agent has never spoken to Joe's agent before, and neither principal wants to manually exchange OAuth credentials to make a 30-minute meeting happen.*

The missing piece is a **trusted venue**: somewhere an agent can show up, present a verifiable credential ("I am acting for Jane Doe, scoped to `meeting.propose`, expires in 1 hour"), discover the counterparty's published skill, run the interaction, and walk away with a signed transcript both principals can audit.

This is the Keybase / Okta layer for agent-to-agent trust, with a meeting room bolted on the side.

---

## What The Lobby is (and isn't)

**Is:**

- An identity broker: signs short-lived agent credentials that prove "this agent is acting for this principal, with these scopes, until this time."
- A discovery surface: resolves `principal@domain` → published skill manifest (an evolution of `agents.md` / A2A agent cards).
- A relay for sessions: brokers the connection, optionally proxies the wire protocol, records a signed transcript.
- Protocol-pluggable: agents speak MCP or A2A over the relay; The Lobby doesn't reinvent tool invocation.

**Is not:**

- A replacement for MCP or A2A. It sits *above* them, providing identity + discovery + audit.
- A general agent-hosting platform. Agents run wherever their principal runs them; The Lobby just gives them a place to meet.
- A model provider. Model-agnostic by construction.

---

## Core concepts

### Principal

A human (or organization) who delegates authority to an agent. Identified by a domain-bound identifier, e.g. `joe@joesoap.com`. Domain ownership is the root of trust — proven via DNS TXT record, well-known endpoint, or an existing IdP federation.

### Agent

A piece of software acting on behalf of exactly one principal at a time. An agent is **not** an identity; it's a delegated actor. The same Claude/GPT/local model can act for different principals in different sessions.

### Skill manifest

The successor to `agents.md`. A signed JSON document at `https://<domain>/.well-known/agent-manifest.json` listing:

- principal identity + key fingerprint
- skills (named tasks the principal accepts)
- per-skill protocol endpoint (MCP server URL, A2A endpoint, etc.)
- per-skill required scopes and rate limits
- routing rules (delegate to other principals for certain skills)

### Delegated credential

Short-lived (default ≤1h), JWT-format, signed by the principal's key. Carries:

- principal DID
- agent fingerprint (which agent instance is allowed to use this)
- scopes (`meeting.propose`, `meeting.book`)
- audience (the counterparty principal)
- expiry, jti
- optional: max budget, max calls, allowed tools

The Lobby never holds these long-term. It mints session credentials on demand from a principal-signed delegation token.

### Session

A bounded interaction between two agents. Has: session ID, both delegated credentials, agreed protocol (MCP or A2A), transcript hash chain, outcome receipt. Both principals get a signed receipt at the end.

---

## Trust model

```
        Principal A (Joe)                   Principal B (Jane)
             │                                     │
     signs delegation                       signs delegation
             │                                     │
             ▼                                     ▼
        Agent A  ──────►   The Lobby   ◄──────  Agent B
                              │
                       verifies both delegations
                       resolves skill manifests
                       opens session, relays calls
                       signs transcript hashes
                              │
                              ▼
                    Signed session receipt
                    (both principals + Lobby)
```

The Lobby is **trusted for liveness and audit, not for authorization.** It cannot mint a credential on a principal's behalf. The worst a malicious Lobby can do is refuse to relay or lie about session existence — it cannot impersonate a principal because it doesn't hold principal keys.

A future version can drop The Lobby from the path entirely once both agents have each other's manifests and credentials, using the Lobby only as a discovery + audit beacon.

---

## Wire flow (calendar example)

1. **Jane asks her agent:** "Book 30 min with Joe Soap next week."
2. **Jane's agent → The Lobby:** `resolve(joe@joesoap.com, skill=meeting-coordination)`. Lobby fetches and verifies `joesoap.com/.well-known/agent-manifest.json`, returns endpoint + required scopes.
3. **Jane's agent → Jane:** "Joe's agent needs scopes `meeting.propose`, `meeting.book` for up to 1h. Approve?" Jane approves; her local key signs a delegation token.
4. **Jane's agent → The Lobby:** `open_session(target=joe@joesoap.com, delegation=<JWT>)`.
5. **The Lobby → Joe's agent:** session offer. Joe's agent presents its own delegation (pre-issued by Joe — Joe doesn't need to be online).
6. **Both agents speak MCP** (or A2A) through the relay. Tool calls and responses are hashed into a Merkle chain.
7. **On `confirm_booking`**, both agents emit a session-end intent. The Lobby produces a signed receipt: `{session_id, participants, transcript_root, outcome, timestamp}`. Both principals' agents store it.
8. **Either principal can later** present the receipt to dispute or audit what their agent agreed to.

---

## Why this is hard (the real engineering)

- **Key custody for principals.** Most humans won't manage signing keys. Need a story: passkey-backed wallet, hosted-but-encrypted, IdP-issued. Probably hosted with hardware-backed keys + recovery, like 1Password for agent identity.
- **Domain proof at scale.** DNS TXT works for technical users; consumer adoption needs IdP federation (Google/Microsoft/Apple sign the principal claim).
- **Replay and confused-deputy attacks.** Delegation tokens must be audience-bound. Sessions must be nonce'd. The Lobby is a tempting MITM target.
- **Liability for agreed actions.** If Joe's agent books a meeting Joe later denies, the signed receipt is the artifact that resolves it. This needs to actually hold up — legal review, not just crypto.
- **Spam and griefing.** A public lobby invites spammy agents. Reputation, rate limits, principal-side deny lists, and proof-of-stake (literal or social) all need answers.
- **Protocol drift.** MCP and A2A are both moving targets. The Lobby has to stay version-tolerant without becoming a translation layer for every dialect.

---

## Repo layout

| Path | Contents |
|---|---|
| [`README.md`](./README.md) | This document — overall design rationale and trust model |
| [`wireframe.html`](./wireframe.html) | Email-signature pattern: dual-track "Book Meeting · 🤖 Agent · 👤 Human" |

More detailed specs (manifest schema, delegation token format, session protocol, threat model) to follow as the design firms up.

---

## Status

Design sketch. No running code. Open to issues, pull requests, and "this already exists, it's called X" corrections — particularly interested in how this overlaps with [A2A](https://github.com/google/A2A), [MCP](https://modelcontextprotocol.io), [NANDA](https://nanda.media.mit.edu/), and DID/Verifiable Credentials work.

## License

MIT.

# The Lobby

> **Pattern 2 of 2.** This repo sketches the *authenticated, sandboxed* version of agent-to-agent meeting coordination. The *public, manual* version lives in [Meet-Me](https://github.com/danielrosehill/Meet-Me). They're two halves of the same idea — these are notes, not a spec.

## The picture

Joe sends Jane an email. The signature ends with a single link:

> 🤖 **Book a meeting (agent only):** `https://lobby.example/s/9c4f...e2a1`

The link is **pre-authenticated, single-use, and ephemeral.** When Jane's agent clicks it:

1. The link drops the agent into a sandboxed session inside The Lobby.
2. The session has Joe's pre-issued grants attached — *read availability for the next 30 days, write exactly one booking, expires in 1 hour.*
3. Jane's agent presents its own delegated credential (signed by Jane, scoped to the same task).
4. The two agents transact inside the sandbox: read availability, propose a slot, confirm a booking.
5. The session closes. Both principals' agents walk away with a signed receipt.

Crucially, the agent **never gets long-lived credentials** to Joe's calendar. The sandbox holds the keys; the sandbox executes the booking; the agents only see what the sandbox is willing to show them.

The intent flow from the recipient's side is dead simple: *agent reads email → spots the link → recognises the pattern → authenticates → books → reports back to its human.*

## Why this is the natural complement to `agents.md`

`agents.md` (Pattern 1) is great at telling another agent *what to do*. It's bad at:

- proving the agent on the other end is who it claims to be,
- authorising the actual write action,
- keeping any of this off the public internet,
- producing an audit trail.

The Lobby fills exactly those gaps. The signature still has a link in it — but that link is a capability, not a public document.

## What The Lobby actually is

A neutral, hosted **sandbox runtime** for short-lived agent-to-agent sessions. Concretely it provides:

- **Pre-authenticated session links.** A principal mints a link tied to a specific skill, specific scopes, an expiry, and (optionally) a target audience. The link encodes a capability; possession of it is the first half of authorisation.
- **An ephemeral session sandbox.** When the link is opened, The Lobby provisions an isolated runtime — fresh state, scoped tools, no access to anything the principal didn't explicitly grant. Think Stripe Checkout or a Firecracker-style microVM, but for an agent conversation.
- **Scoped tool execution.** The sandbox exposes only the tools needed for the task (`read_availability`, `propose_slot`, `confirm_booking`). It speaks MCP or A2A on the wire so existing agent runtimes can talk to it.
- **Signed transcripts.** Every tool call inside the session is hashed into a chain. At session close, both principals get a signed receipt: who showed up, what they agreed to, when.

## What it isn't

- Not a replacement for MCP or A2A. The Lobby uses them as the wire protocol.
- Not a model provider or agent host. Agents run wherever their principal runs them.
- Not a public directory of agents. The interaction surface is per-session, capability-gated.

## The hard parts (open questions)

This is a work-in-progress idea, not a finished design. The genuinely unresolved bits:

- **What's the sandbox runtime?** Something like this almost certainly already exists adjacent — sandboxed code execution (E2B, Modal, Cloudflare Workers / Durable Objects, Fly Machines), browser sandbox agents (Browserbase, Steel), or session-scoped capability runtimes. Most likely the right move is to compose what's already there rather than build a new sandbox primitive. **Looking for prior art** — if this is already a product, please tell me.
- **Key custody for principals.** Most humans won't manage signing keys. Probably hosted-but-recoverable, passkey-anchored.
- **Domain proof at scale.** DNS TXT works for technical users; consumer adoption needs IdP federation (Google / Microsoft / Apple sign the principal claim).
- **Replay and confused-deputy attacks.** Capability links must be audience-bound and one-shot; sandboxes must be nonced; The Lobby is itself a juicy MITM target.
- **Liability for what the agent agreed to.** The signed receipt is the artifact that resolves disputes — needs legal review, not just crypto.
- **Spam, griefing, link harvesting.** A pre-authenticated link in an email signature is still a URL someone could exfiltrate. Single-use + audience-bound + short expiry caps the damage, but the threat model needs proper work.

## Who would actually use this

The pain point is mundane and universal: scheduling ping-pong, Calendly link emails, group bookings that take fifteen messages, "let me check with my team" threads that stall for days. None of this needs a new social network or a model breakthrough — it just needs a place where two agents can meet, prove identity, do one bounded thing, and leave.

If agents reading email on their humans' behalf become common — which feels likely — then the dual-track signature pattern (one link for humans, one capability link for agents) becomes a natural idiom.

## What's in this repo

| File | What it is |
|---|---|
| [`README.md`](./README.md) | This document — the idea and the open questions. |
| [`wireframe.html`](./wireframe.html) | The dual-track signature wireframe: `Book Meeting · 🤖 Agent · 👤 Human`. |

Specs (manifest schema, capability-link format, session protocol, threat model) will follow if/when the design firms up.

## Status

Notes. An open loop. I haven't done a full prior-art sweep — there's likely overlap with [A2A](https://github.com/google/A2A), [MCP](https://modelcontextprotocol.io), [NANDA](https://nanda.media.mit.edu/), DID/Verifiable Credentials, and any number of existing sandbox runtimes. If you're working on something close, I'd love to know.

## License

MIT.

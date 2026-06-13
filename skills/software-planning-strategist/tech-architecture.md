# Tech Architecture Reference

Use this file during Phase 5 of the software planning workflow to guide stack selection and system design.

---

## Stack Selection Decision Trees

### Frontend

| Signal | Recommendation |
|--------|---------------|
| Consumer-facing web app, SEO matters | Next.js (React, SSR/SSG) |
| Internal tool / dashboard, no SEO needed | React + Vite, or SvelteKit |
| Mobile-first required | React Native (cross-platform) or Swift/Kotlin (native performance critical) |
| Real-time, highly interactive (e.g., collaborative editing) | SvelteKit or SolidJS (performance-first) |
| Rapid prototype / MVP with minimal frontend work | Next.js + shadcn/ui + Tailwind |

### Backend

| Signal | Recommendation |
|--------|---------------|
| API-first, fast iteration, small team | Node.js + Express or Fastify / Python + FastAPI |
| High-throughput, low-latency microservices | Go or Rust |
| ML/AI-heavy processing | Python (FastAPI + Celery for async jobs) |
| Enterprise, strong typing requirements | TypeScript (Node) or Java (Spring Boot) |
| Serverless / event-driven | AWS Lambda + API Gateway / Vercel Edge Functions |

### Database

| Data Model | Recommendation | Avoid If |
|------------|---------------|----------|
| Relational, complex joins | PostgreSQL | Schema changes very frequent |
| Document, flexible schema | MongoDB | Data is highly relational |
| Real-time sync (e.g., mobile) | Firestore / Supabase | Cost at scale is prohibitive |
| Time-series (IoT, analytics) | TimescaleDB / InfluxDB | Not time-series data |
| Graph (social networks, recommendations) | Neo4j / Amazon Neptune | Graph patterns aren't core |
| Key-value / cache | Redis | Persistence is critical primary store |
| Search | Elasticsearch / Algolia | Simple keyword search only |

> **Default for most apps**: PostgreSQL + Redis (cache/queue). Only deviate with strong justification.

### Auth

| Use Case | Recommendation |
|----------|---------------|
| Fast MVP, don't want to build auth | Clerk, Auth0, Supabase Auth |
| Full control, on-prem possible | NextAuth.js (self-hosted) |
| Enterprise SSO required | Auth0 or Okta |
| Mobile app | Firebase Auth or Cognito |

### Infrastructure

| Stage | Recommendation |
|-------|---------------|
| MVP / Prototype | Vercel (frontend) + Railway or Render (backend) + Supabase (DB) — cheapest, fastest |
| Growth stage | AWS (ECS/EKS) or GCP — more control, more complexity |
| Enterprise | AWS or Azure with Terraform IaC |
| AI/ML workloads | AWS SageMaker / GCP Vertex / Modal for GPU inference |

---

## System Design Patterns by Product Type

### SaaS B2B Product
- Multi-tenancy: Row-level security in PostgreSQL (tenant_id on every table)
- Auth: SSO support from day one (even if not used until enterprise)
- Billing: Stripe + usage metering (Stripe Meters or custom)
- Audit logs: Every user action written to an immutable log table
- Rate limiting: Per-tenant, not global
- Key consideration: Data isolation guarantees are a sales requirement

### Consumer Mobile App
- Offline-first: Local state with sync (SQLite + sync layer or Realm)
- Push notifications: Firebase Cloud Messaging
- Analytics: Mixpanel or Amplitude (event-based, not just pageviews)
- App store compliance: Review cycles are 1–3 days; plan for this
- Key consideration: Day-1, Day-7, Day-30 retention is everything

### Marketplace (Two-sided)
- Hardest problem: Cold start (chicken-and-egg)
- Supply side almost always needs to come first
- Payments: Stripe Connect for split payments
- Trust & safety: Review system, dispute flow, and fraud detection needed early
- Key consideration: Define network effects mechanism before building

### AI-Powered App
- LLM calls are expensive: Cache aggressively (semantic caching with Redis)
- Latency: Streaming responses are mandatory for UX
- Prompt management: Treat prompts as code (version control, A/B test)
- Fallback: Always have a non-AI fallback path
- Observability: Log every LLM input/output for debugging and cost analysis
- Key consideration: LLM providers go down; build retry logic and provider fallback

### Real-time Collaboration Tool
- WebSockets: Socket.io (quick) or raw WS with Redis pub/sub (scale)
- Conflict resolution: CRDTs (Yjs, Automerge) for true concurrent editing
- Presence: Who's online, where are they in the document
- Key consideration: Operational transforms vs. CRDTs — decide before day one, it's very hard to change

---

## Scalability Thinking

### What breaks first at 10x load?

Ask this for every architectural decision:

1. **Database**: Connection pooling (use PgBouncer), N+1 queries, missing indexes
2. **API**: Single-threaded blocking calls, synchronous heavy operations
3. **File storage**: Never store files in a database — use S3/GCS from day one
4. **Sessions**: Stateful server sessions don't scale horizontally — use JWTs or Redis
5. **Background jobs**: Synchronous jobs that should be async (email, notifications, reports)
6. **Third-party rate limits**: Stripe, Twilio, LLM providers all have limits

### Premature Optimization Warning
Do NOT over-architect for scale that doesn't exist yet. Rule of thumb:
- MVP: Optimize for developer velocity
- 1K users: Optimize for correctness and reliability
- 10K users: Optimize for performance hotspots (measure first)
- 100K users: Optimize for scalability (microservices, caching layers)

---

## Security Checklist (Flag non-compliance in Phase 5.3)

- [ ] All secrets in environment variables (never committed)
- [ ] HTTPS only (HSTS header)
- [ ] Input validation on ALL user inputs (never trust client)
- [ ] Parameterized queries (never string-interpolate SQL)
- [ ] Auth on every protected API route (no security by obscurity)
- [ ] Rate limiting on auth endpoints
- [ ] CORS configured properly
- [ ] PII data encrypted at rest
- [ ] Dependency scanning (Snyk, Dependabot)
- [ ] OWASP Top 10 awareness

---

## Regulatory Flags (Raise if applicable)

| Domain | Regulation | Impact |
|--------|-----------|--------|
| Health data (US) | HIPAA | BAA required, audit logs, encryption mandatory |
| EU users | GDPR | Data residency, right to erasure, consent flows |
| Financial services (US) | PCI-DSS / SEC / FINRA | Don't store card data; licensing may be required |
| Children's apps | COPPA | No behavioral tracking under 13 |
| Accessibility | WCAG 2.1 AA | Legal risk if public-facing in many jurisdictions |

Always flag these in Phase 5.3 if the product touches any of these domains.

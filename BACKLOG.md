# Oricode AI - Product Backlog

*Last updated: January 31, 2026*

## Architecture

```
Supabase (PostgreSQL) → Railway (oricode-backend) → Clients
                                                     ├── Eclipse plugin (oricode-poc)
                                                     ├── VS Code extension (oricode-vs-code)
                                                     ├── oricode-app (Vercel) - Dashboard
                                                     └── oricode-landing (Vercel) - Website
```

**Backend endpoints:** /api/chat, /api/chats, /api/mcp/*, /api/user, /api/stripe, /api/credits, /api/admin

**Shared code:** Centralized composables across frontend apps

---

## 🔴 P0 - Launch Blockers

| # | Task | Est. | Status |
|---|------|------|--------|
| 0.1 | Fix landing page CTAs → `app.oricode.ai` | 15 min | ✅ Done |
| 0.2 | Privacy Policy page | 30 min | ✅ Done |
| 0.3 | Terms of Service page | 30 min | ✅ Done |
| 0.4 | Test billing E2E (Stripe test mode) | 15 min | ✅ Done |
| 0.5 | Plugin download link on website | 15 min | ✅ Done |

**🎉 ALL P0 BLOCKERS COMPLETE - READY TO GO LIVE!**

---

## 🎯 WHAT TO FOCUS ON NOW

### Immediate Priority (This Week)
| Task | Why | Est. |
|------|-----|------|
| **2.5 RAP OData V4 investigation** | Currently in progress, finish it | 3-4 hr |

### Business Blockers (Waiting)
| Item | Status | Action Needed |
|------|--------|---------------|
| Bank Account | ⏳ Mercury under review | Wait for approval |
| Stripe Live Mode | ⏳ After bank | Switch from test to live keys |

### Recommended Focus Order
1. **Finish RAP OData V4** → unlocks new use cases for customers

---

## 🟠 P1 - Security (2FA)

| # | Task | Est. | Status |
|---|------|------|--------|
| 1.1 | Email verification on signup | 1.5 hr | ✅ Done (Jan 28) |
| 1.2 | Authenticator app (TOTP) | 2 hr | ✅ Done |
| 1.3 | Passkey (WebAuthn) | 2-3 hr | ⬜ |
| 1.4 | SMS/Phone verification | 2 hr | ⬜ |

**Note:** TOTP 2FA fully implemented with QR code setup, backup codes, and admin 2FA requirement. Email verification sends on signup with 24hr token expiry.

---

## 🟠 P1 - Platform Expansion

| # | Task | Est. | Status |
|---|------|------|--------|
| 1.5 | VS Code / BAS extension | 8-12 hr | ✅ Done (v1.0.0) |
| 1.6 | Help pages for oricode-app | 4 hr | ✅ Done (Jan 28) |
|     | - Eclipse installation & setup | | ✅ |
|     | - VS Code extension guide | | ✅ |
|     | - Subscription management | | ✅ |
|     | - 2FA setup | | ✅ |
|     | - API key management | | ✅ |
|     | - SAP connection configuration | | ✅ |

**Note:** VS Code extension available at `/downloads/oricode-ai-1.0.0.vsix`. Help center at `/help` with 6 topic pages.

---

## 🟡 P2 - Plugin Improvements

| # | Task | Est. | Status |
|---|------|------|--------|
| 2.1 | MCP lock handling for CRUD | 1-2 hr | ✅ Done |
| 2.2 | Fix project selection bug | 30 min | ✅ Done |
| 2.3 | Update logo ⬡ | 20 min | ✅ Done (orange hexagon) |
| 2.4 | Local class generation fix (lcl_* classes) | 2 hr | ✅ Done (Jan 21) |
|     | - Correct system prompt for lcl_* classes | | ✅ |
|     | - Ensure proper include_type handling | | ✅ |
| 2.5 | RAP OData V4 investigation | 3-4 hr | 🔍 In Progress |
|     | - ADT-based OData service creation | | ⬜ |
|     | - System prompt style for RAP OData V4 generation | | ⬜ |
| 2.6 | Code obfuscation | 2-3 hr | ✅ Done (Jan 28) |
|     | - Eclipse plugin (ProGuard) | | ✅ |
|     | - VS Code extension (webpack + terser) | | ✅ |

---

## 🟢 P3 - Nice to Have

| # | Task | Est. | Status |
|---|------|------|--------|
| 3.1 | Social login (Google, GitHub, Apple) | 4-5 hr | ⬜ |
| 3.2 | Team management | 4+ hr | ⬜ |
| 3.3 | Chat history search / RAG context | 4 hr | ⬜ |

---

## 🔵 P4 - Enterprise / On-Prem

| # | Task | Est. | Status |
|---|------|------|--------|
| 4.1 | CodeLlama / On-Prem LLM support | 6-8 hr | ⬜ |
| 4.2 | SSO/SAML integration | 4-6 hr | ⬜ |
| 4.3 | Air-gapped deployment docs | 2-3 hr | ⬜ |
| 4.4 | License system backend API | 4-6 hr | ⬜ |
|     | - POST /api/license/generate | | ⬜ |
|     | - GET /api/license/download | | ⬜ |
|     | - Stripe webhook integration | | ⬜ |
| 4.5 | License dashboard UI (oricode-app) | 3-4 hr | ⬜ |
|     | - My Licenses page | | ⬜ |
|     | - Download button | | ⬜ |
|     | - Regenerate if lost | | ⬜ |
| 4.6 | Rust MCP rewrite (maximum protection) | 2-3 weeks | ⬜ |
|     | - Rewrite adt-client.ts in Rust | | ⬜ |
|     | - Native binary (harder to reverse) | | ⬜ |
|     | - Smaller binary (~5MB vs 37MB) | | ⬜ |
| 4.7 | Environment landscape (dev/test/prod) | 3-4 hr | ⬜ |
|     | - Currently only production environment exists | | ⬜ |
|     | - Set up separate dev environment for testing | | ⬜ |
|     | - Set up staging/test environment | | ⬜ |
|     | - Environment-specific configs (DATABASE_URL, API keys) | | ⬜ |

**Note:** MCP server now protected with bytecode + license validation (Jan 31, 2026). Rust rewrite would provide maximum protection against reverse engineering.

---

## ✅ Completed

### Infrastructure & Backend
| Task | Date |
|------|------|
| MCP Tools via SaaS | Jan 2026 |
| Railway deployment + CICD | Jan 2026 |
| PostgreSQL / Supabase setup | Jan 2026 |
| Credit system (balance, usage, packages) | Jan 2026 |
| API key authentication | Jan 2026 |
| Chat persistence | Jan 2026 |
| Prompt caching (90% savings) | Jan 2026 |
| Rate limiting per plan (free/pro/enterprise) | Jan 2026 |
| Token usage tracking (monthly) | Jan 2026 |

### Stripe & Billing (FULLY IMPLEMENTED)
| Task | Date |
|------|------|
| Stripe subscription checkout (monthly/yearly) | Jan 2026 |
| Stripe credit purchases ($10/$50/$100 packages) | Jan 2026 |
| Stripe webhook processing (with retry logic) | Jan 2026 |
| Stripe billing portal integration | Jan 2026 |
| Subscription status sync (active/canceling/canceled) | Jan 2026 |
| Admin revenue dashboard & MRR tracking | Jan 2026 |

### Clients & Apps
| Task | Date |
|------|------|
| Eclipse plugin core | Jan 2026 |
| VS Code extension (v1.0.0 with chat history) | Jan 2026 |
| oricode-app dashboard (Nuxt 3 + Vercel) | Jan 2026 |
| oricode-landing website (Nuxt 3 + Vercel) | Jan 2026 |
| Centralized composables | Jan 2026 |

### Dashboard Features (oricode-app)
| Task | Date |
|------|------|
| Full JWT authentication | Jan 2026 |
| 2FA with TOTP (QR + backup codes) | Jan 2026 |
| Role-based access (User/Admin/SuperAdmin) | Jan 2026 |
| API key management (show/copy/regenerate) | Jan 2026 |
| Subscription management UI | Jan 2026 |
| Credit purchase UI | Jan 2026 |
| Usage tracking page | Jan 2026 |
| Admin user management + Excel export | Jan 2026 |
| Admin income analytics | Jan 2026 |
| Theme toggle (light/dark/system) | Jan 2026 |
| Password reset flow | Jan 2026 |
| Account deletion with confirmation | Jan 2026 |

### Landing Website (oricode-landing)
| Task | Date |
|------|------|
| Landing page with hero, features, pricing | Jan 2026 |
| Privacy Policy page | Jan 6, 2026 |
| Terms of Service page | Jan 6, 2026 |
| Plugin download links (Eclipse + VS Code) | Jan 2026 |
| All CTAs pointing to app.oricode.ai | Jan 2026 |
| Beta page for testers | Jan 2026 |

### Plugin Improvements
| Task | Date |
|------|------|
| Local class deployment fix (include_type param) | Jan 21, 2026 |
| Timeout prevention (40% → 3% rate) | Jan 21, 2026 |
| File-based chunking implementation | Jan 22, 2026 |
| System prompt optimization (76% reduction) | Jan 21, 2026 |
| Pill button state persistence | Jan 21, 2026 |
| Smart interleaved workflow | Jan 21, 2026 |
| Max tokens increase (4K → 8K) | Jan 21, 2026 |
| ABAP Cloud browser auth support | Jan 23, 2026 |

### Business
| Task | Date |
|------|------|
| Delaware LLC formation (File #10458040) | Jan 2026 |
| EIN received (38-4380624) | Jan 28, 2026 |

### Security Features
| Task | Date |
|------|------|
| Code obfuscation (ProGuard + webpack/terser) | Jan 28, 2026 |
| Email verification on signup | Jan 28, 2026 |
| Help center (6 pages) | Jan 28, 2026 |
| MCP bytecode protection (pkg --bytecode) | Jan 31, 2026 |
| RSA license signing system | Jan 31, 2026 |
| License validation in MCP server | Jan 31, 2026 |

---

## 📊 Business Status

| Item | Status |
|------|--------|
| Company | Oricode LLC (Delaware, File #10458040) |
| Formation | ✅ Complete |
| ID Verification | ✅ Complete |
| EIN | ✅ 38-4380624 (Jan 28, 2026) |
| Bank Account | ⏳ Mercury under review |
| Stripe Payments | ⏳ Ready to activate |

### Anthropic Partnership

| Item | Status |
|------|--------|
| Dev account (3a64254a-aa08-40e0-8f2f-1b670bc2972f) | ✅ Tier 4 (Jan 5, 2026) |
| Create dedicated Oricode Org ID | ⬜ |
| Request Tier 4 for Oricode Org | ⬜ Send to Claire |
| Ask about Tier 5 / custom limits for production | ⬜ |
| Contact: Claire Noemer (cnoemer@anthropic.com) | |

---

## 🎯 Release Roadmap

### v1.0 - MVP ✅ COMPLETE
- [x] Eclipse plugin with MCP CRUD
- [x] Credit system
- [x] Chat persistence

### v1.1 - Launch Ready ✅ COMPLETE
- [x] Website live (oricode-landing on Vercel)
- [x] Stripe subscriptions active (Pro/Enterprise, monthly/yearly)
- [x] VS Code extension available
- [ ] Plugin on Eclipse Marketplace

### v1.5 - Security ✅ MOSTLY COMPLETE
- [x] 2FA TOTP (authenticator apps)
- [x] Email verification on signup
- [x] Help pages (6 topics)
- [ ] 2FA WebAuthn (passkeys)

### v2.0 - Platform
- [x] VS Code extension ✅
- [ ] Web chat interface
- [ ] Team features

### v3.0 - Enterprise
- [ ] SSO/SAML
- [ ] On-prem LLM
- [ ] Audit logs (admin audit logging already exists)

---

## 📈 Pricing & Success Metrics

### Current Pricing
| Plan | Monthly | Yearly | Credits |
|------|---------|--------|---------|
| Free | $0 | - | 100/month |
| Pro | $35/mo | $29/mo ($348/yr) | 2,000/month |
| Enterprise | $119/mo | $99/mo ($1,188/yr) | 5,000/month |

### Credit Packages (One-time)
| Credits | Price |
|---------|-------|
| 300 | $10 |
| 1,500 | $50 |
| 3,500 | $100 |

### Success Metrics (Targets)

| Metric | 6 months | 1 year |
|--------|----------|--------|
| Active Users | 500 | 5,000 |
| Paying Customers | 50 | 500 |
| MRR | $2,500 | $25,000 |

---

## 💰 Anthropic API Projections

| Period | Spend |
|--------|-------|
| Year 1 | $150K-$250K |
| Year 2 | $800K-$1.2M |
| Year 3 | $2M-$3M |

**Current limits needed:**
- Development: 100K input/output tokens/min
- Production: 500K tokens/min
- Scale: 1M+ tokens/min

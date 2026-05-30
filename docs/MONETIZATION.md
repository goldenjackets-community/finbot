# FinBot — Monetization Strategy

## Business Models (under evaluation)

### Option A: B2C Direct (current plan)
- **Price:** R$19,90/month per user
- **Target:** Individuals who want to manage finances via WhatsApp
- **Pros:** Large market, simple onboarding
- **Cons:** High churn, needs volume (500+ paying users for R$10K/month), expensive acquisition
- **Break-even:** ~50 paying users covers AWS costs

### Option B: B2B (sell to companies)
- **Price:** R$5-10/employee/month
- **Target:** HR departments, benefits companies, fintechs
- **Pros:** 1 contract = 100-500 users, high retention (company pays), predictable revenue
- **Cons:** Longer sales cycle, needs customization per client
- **Break-even:** 1 company with 200 employees = R$2.000/month

### Option C: White-Label
- **Price:** R$500-2.000/month per client
- **Target:** Financial coaches, accountants, fintechs with existing audience
- **Pros:** They bring the users, we provide the tech. High margin.
- **Cons:** Needs multi-tenant architecture, support overhead
- **Break-even:** 3-5 white-label clients = R$3.000-10.000/month

### Option D: Hybrid (recommended)
- Launch as B2C (validates product, builds portfolio)
- Track retention metrics from day 1
- If retention > 40% at 30 days → scale B2C
- If retention < 40% → pivot to B2B or white-label with same codebase

## Revenue Maximization Tactics

### From Day 1
- [ ] Landing page with waitlist (validate demand before building everything)
- [ ] Trial model: 7 days free, then R$19,90/month (not freemium)
- [ ] Payment integration: Stripe or Mercado Pago
- [ ] Onboarding asks for payment method upfront

### Metrics to Track
- [ ] DAU/MAU ratio (daily active / monthly active)
- [ ] Retention at 7, 14, 30 days
- [ ] Messages per user per day
- [ ] Conversion: free trial → paid
- [ ] Churn rate (monthly)
- [ ] CAC (customer acquisition cost)
- [ ] LTV (lifetime value)

### DynamoDB Tracking (add to single-table)
| Entity | PK | SK | Purpose |
|--------|----|----|---------|
| Metrics | `USER#<phone>` | `METRIC#last_active` | Retention tracking |
| Metrics | `USER#<phone>` | `METRIC#messages_count` | Engagement |
| Subscription | `USER#<phone>` | `SUB#active` | Payment status |

## Competitive Advantage

| Us | Meu Planner Financeiro (37K users) |
|----|------------------------------------|
| AI conversational (real dialogue) | Basic registration only |
| WhatsApp native (no app download) | Web responsive + PWA |
| Voice/photo input (future) | Text only |
| Predictive insights | Historical only |
| R$19,90/month | R$16,50/month |

## Decision Deadline

**Sunday 8:00 AM (kickoff)** — Final decision on which model to pursue for MVP.

MVP code is the same regardless of model. The difference is:
- Landing page copy
- Onboarding flow (trial vs free)
- Who we target first

## Notes
- Code is model-agnostic — same Lambda/DynamoDB/Bedrock stack works for B2C, B2B, or white-label
- Multi-tenant can be added later (PK includes tenant ID)
- Payment integration is a separate spec (not in MVP week 1-4, add in week 5-6)
", "path": "docs/MONETIZATION.md", "message": "docs: add monetization strategy with B2C/B2B/white-label options", "owner": "goldenjackets-community", "repo": "finbot", "sha": null}
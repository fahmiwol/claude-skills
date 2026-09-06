# Tiranyx Claude Skills

> Battle-tested Claude Code skills extracted from real production work — Meta ads tracking, multi-tenant Next.js, Shopee data import, and domain verification.

Curated by [@fahmiwol](https://github.com/fahmiwol) (Founder, PT Tiranyx Digitalis Nusantara) — built while shipping [TWKCard](https://twkdg.id), an Indonesian biolink + flip-card platform.

Each skill is a single self-contained `SKILL.md` with copy-pasteable code, gotchas-as-tables, and verification commands. No fluff, no AI marketing copy. Just things that worked.

---

## 📦 Skills

| Skill | What it does | Lines |
|---|---|---|
| **[meta-capi-dedup-setup](./meta-capi-dedup-setup/SKILL.md)** | Wire up Meta Pixel + Direct Conversions API with browser↔server `event_id` dedup, server-side `_fbp`/`_fbc` generation, `external_id` hashing, and PageView/ViewContent/AddToCart mirroring. EMQ-optimized, no Gateway needed. | 316 |
| **[facebook-domain-verification](./facebook-domain-verification/SKILL.md)** | Verify a domain in Facebook Business Manager using BOTH the HTML file method AND the meta-tag method simultaneously, with Next.js middleware exemption for site verification files. | 139 |
| **[nextjs-subdomain-multitenant](./nextjs-subdomain-multitenant/SKILL.md)** | Subdomain → brand/tenant rewriting in Next.js 15 App Router with public file exemptions, dev-mode localhost fallback, and apex landing pass-through. | 188 |
| **[shopee-data-import](./shopee-data-import/SKILL.md)** | Brutally honest decision tree for importing product data from a Shopee storefront. Documents what works (CSV export, Meta Catalog API, manual seed) and what fails (raw HTTP, bare Playwright, cookie injection). | 190 |
| **[shopee-open-platform-integration](./shopee-open-platform-integration/SKILL.md)** | Build a Shopee Open Platform "Partner App" so ONE developer account onboards MANY seller shops via OAuth. HMAC-SHA256 signing, 4h/30d token lifecycle, minimum-scope strategy for faster production review. The legitimate alternative to `shopee-data-import` when you have agency/SaaS scale. | 280 |

---

## 🚀 Install

### Per-user (global, available in every project)

Clone into your `~/.claude/skills/` directory:

```bash
# Windows (Git Bash / WSL)
cd "$USERPROFILE/.claude/skills"
git clone https://github.com/fahmiwol/claude-skills.git tiranyx
ln -s tiranyx/meta-capi-dedup-setup ./meta-capi-dedup-setup
ln -s tiranyx/facebook-domain-verification ./facebook-domain-verification
ln -s tiranyx/nextjs-subdomain-multitenant ./nextjs-subdomain-multitenant
ln -s tiranyx/shopee-data-import ./shopee-data-import

# macOS / Linux
cd ~/.claude/skills
git clone https://github.com/fahmiwol/claude-skills.git tiranyx
for s in meta-capi-dedup-setup facebook-domain-verification nextjs-subdomain-multitenant shopee-data-import; do
  ln -s tiranyx/$s ./$s
done
```

Restart Claude Code. Skills should show up via auto-discovery — try `/meta-capi-dedup-setup` to verify.

### Per-project

Copy the skill folder into your project's `.claude/skills/`:

```bash
cd /path/to/your-project
mkdir -p .claude/skills
cp -r /path/to/this-repo/meta-capi-dedup-setup .claude/skills/
```

---

## 🧠 Why these skills?

Claude Code skills are markdown files Claude can auto-discover and inject as expertise into your session. When you mention "Meta Pixel" or "Facebook domain verification" in any project, Claude picks up the relevant skill and follows the playbook — instead of guessing or going to docs.

Each skill here distills **multiple days of debugging** into a 100-300 line markdown file. Specifically:

- **`meta-capi-dedup-setup`** — From scratch, getting browser↔server dedup right (event_id, _fbp generation, EMQ tuning) took several iterations across Meta docs + Events Manager testing. The skill captures the working pattern + the gotchas table.
- **`facebook-domain-verification`** — The HTML file method silently fails on Next.js subdomain rewrites because middleware swallows the file. The skill captures the middleware regex exemption.
- **`nextjs-subdomain-multitenant`** — Every multi-tenant SaaS rebuilds this from scratch. This is the rewriter that handles `acme.app.com/api/*` (apex pass-through) + `acme.app.com/` (rewrite to `/acme`) + `app.com/` (apex landing) + `acme.localhost:3000` (dev convenience) — with public file exemptions.
- **`shopee-data-import`** — Saves you from going down the Playwright stealth rabbit hole. Shopee anti-bot is harder than you think. The skill ranks 5 paths by reliability and tells you which one to actually use.

---

## 🛠️ Adding your own skill

Skills are dead simple. Create a folder + a `SKILL.md` with YAML frontmatter:

```markdown
---
name: my-skill-name
description: Concise one-paragraph description. When should Claude trigger this skill? What problem does it solve? Be specific — vague descriptions trigger noise.
---

# My Skill Title

Body content. Whatever helps Claude reproduce the pattern.
```

Drop the folder into `~/.claude/skills/my-skill-name/` and restart Claude Code.

Better, more triggering descriptions get auto-invoked more often. Look at how this repo writes descriptions — list **exact triggering phrases** and **the failure modes the skill prevents**.

---

## 🤝 Contributing

PRs welcome! Especially:
- Region-specific anti-bot lessons for SEA e-commerce sites (Tokopedia, Lazada, Blibli)
- More Meta CAPI patterns for niche events (Purchase, CompleteRegistration, custom events)
- Alternative subdomain routing patterns (Cloudflare Workers, edge middleware)

Open an issue first if it's a new skill — we want signal-to-noise high.

---

<!--toko-mulai-->

## Related tools

More of the same working method, extracted from real sessions:

- **[Agent Memory Starter](https://github.com/fahmiwol/agent-memory-starter)** — free, open source. Stop re-explaining your project every session.
- **[Agent Memory OS](https://fahmiwolf.gumroad.com/l/npfhry)** — $5. The full method.
- **[Second Brain Kit](https://fahmiwolf.gumroad.com/l/ezqudk)** — $7. The same idea as an MCP server your agent queries.

All of them: [fahmiwolf.gumroad.com](https://fahmiwolf.gumroad.com)

<!--toko-akhir-->

## 📄 License

MIT. Use these freely in commercial or open-source projects. Attribution appreciated but not required.

If a skill saves you a billable day, [send Fahmi a coffee](https://saweria.co/fahmiwol) or just tell other devs to try it.

---

## 🔗 Related

- [Claude Code](https://claude.com/code) — the agentic CLI that loads these skills
- [Anthropic Skills](https://docs.claude.com/en/docs/claude-code/skills) — official skill docs
- [TWKCard](https://twkdg.id) — biolink + flip-card platform these skills were extracted from

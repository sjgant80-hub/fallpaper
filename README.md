# FallPaper · sovereign FCA-shaped document generator

Single-file tool that generates the documents a UK financial-advisory firm has to produce to stay compliant: engagement letters, fact-finds, suitability reports, annual review packs, MiFID II costs disclosure, CIDD, vulnerable-customer assessment notes and tax-year-end letters.

Runs entirely in the browser. Client data never leaves the device. IndexedDB. Sovereign.

**Live:** https://sjgant80-hub.github.io/fallpaper/

---

## For end users · advisers and paraplanners

**What it does.** Pick a client. Pick a template. Click Generate. You get an FCA-shaped document with the client's data interpolated and the regulatory clauses already in place.

**Eight templates, all included:**

| Template | Regulator hook | What it's for |
|---|---|---|
| Engagement Letter / Client Agreement | COBS 6.1 | First meeting · scope, fees, complaints, FOS, FSCS, cancellation rights |
| Client Fact-Find | COBS 9.2 (KYC) | Discovery document · identity, income, assets, ATR/CFL/K&E, vulnerability |
| Suitability Report | COBS 9.4 | Reasoned recommendation · required before any retail investment transaction |
| Annual Review Pack | COBS 9.4.10R | Ongoing suitability · performance vs benchmark · ex-post costs |
| MiFID II Costs & Charges | COBS 6.1ZA | Ex-ante or ex-post itemised cost disclosure with RIY |
| CIDD (Client Investment Disclosure) | KFD / firm disclosure | Plain-English firm summary issued before engagement |
| Vulnerable Customer Note | FCA FG21/1 | Internal evidence of identification + accommodation |
| Tax Year-End Letter | service touch | Late-Feb prompt to use ISA / pension / CGT allowances |

**Regulatory clauses are locked.** The complaints text, FSCS limits, cancellation rights, cooling-off periods, risk warnings, MiFID disclosure framing and FG21/1 framing all carry a lock and can't be edited — they're the bits the FCA cares about. Everything else is yours to edit.

**Lives inside a bundle.** FallPaper listens on the `fall-client` BroadcastChannel:
- When **FallOnboard** captures a client, FallPaper sees it instantly.
- When **FallAdviser** runs a scenario, you can pull it in and pre-fill the suitability report.
- When **FallPractice** posts a fee, it shows up in the costs section.

If those tools aren't open, FallPaper still works on its own.

**Three export paths:**
- Markdown (for filing or copying into your back-office)
- Standalone HTML (open and print)
- Hand-off to **FallPDF** (postMessage; if FallPDF is open in another tab, it'll render the PDF)

**Disclaimer.** Templates are guidance, not compliance certification. The firm's compliance officer remains responsible for accuracy and completeness. Sovereign — client data never leaves the device.

### First time · five-minute setup

1. Open the live URL or `index.html` from disk.
2. **Firm tab** → enter your firm name, FCA reference, registered address. Click save.
3. **Firm tab → Advisers** → add yourself as the active adviser.
4. **Clients tab** → click + new client → fill in the basics (or wait for FallOnboard to push one).
5. **Generate tab** → pick the client, pick a template, hit commit + save.

The demo data (Marcus Osei · DEMO Wealth Ltd) is overwriteable — just edit the firm record with your real details.

### WhatsApp paste

> FallPaper · sovereign FCA document generator. 8 templates: engagement letter, fact-find, suitability, annual review, MiFID costs, CIDD, vulnerable-customer note, tax year-end. Client data stays on your device. Live: https://sjgant80-hub.github.io/fallpaper/

---

## For developers

**Architecture.** One HTML file, vanilla JS, no build step. ~130 KB. The deliverable is `index.html`.

```
fallpaper/
├── index.html      # the deliverable · all logic, all CSS, all 8 templates
├── README.md       # this file
├── LICENSE         # MIT
└── .nojekyll       # tells GitHub Pages not to Jekyll-process the repo
```

**Stack.**
- Vanilla JS, no framework
- IndexedDB stores: `firms`, `advisers`, `clients`, `documents`, `templates`, `audit`, `state`
- BroadcastChannel `fall-client` (shared schema mesh) + `fall-signal` (low-level handshake)
- PWA manifest via `data:` URL — installable
- KONOMI shim (sovereign tier, inert)
- `window.FALLPAPER` exposed for in-browser debug / verify protocol

**Shared client schema.** FallPaper conforms exactly to `IFA-BUNDLE-SHARED-SCHEMA.md` v1.0. Client / Adviser / Firm record shapes are identical across the IFA bundle (FallAdviser v2, FallOnboard, FallPaper, FallPractice). Writes broadcast the full new record (not a diff) with 300ms debounce. Boot emits `sync.request`; other tools respond with `sync.snapshot`.

**Template definition.** Every template is a JS object in `TEMPLATES_BUILTIN`:

```javascript
{
  id: 'suitability',
  name: 'Suitability Report',
  version: '1.0',
  cobs: 'COBS 9.4',
  kind: 'recommendation',
  description: '...',
  sections: [
    { id: 'header', heading: '...', body: '{{client.firstName}} ...' },
    { id: 'risks',  heading: '...', body: '...', locked: true },
    // ...
  ]
}
```

**Interpolation.** `{{path.to.field}}` walks the rendering context (client, firm, adviser, suitability, engagement, recommendation, review, ctx, today, docRef, taxYear). Missing values render as `<span class="placeholder-empty">{{path}}</span>` — visible warning, not silent break.

**Custom sections.** The Template editor lets the firm override the body of unlocked sections. Overrides save to the `templates` IDB store keyed by template id. Locked (regulatory) sections refuse edits.

**Audit chain (Mansoor P3).** Every state-changing action appends to `audit`:
- `i`: sequential index
- `prevHash`: sha256 of the previous entry's docHash
- `docHash`: sha256 of `{i, ts, action, clientId, prevHash, payload}`
- `configVersion`: `fallpaper@1.0.0`
- `reasoning`: human-readable why

Exportable as JSON. Verifiable from the Audit tab.

**T0 / T3 cascade.**
- **T0** offline keyword router · 12 canned answers about doc types, COBS rules, FOS/FSCS limits, ATR/CFL/K&E, FG21/1, cancellation rights. Zero network.
- **T3** BYOK fallback · uses an Anthropic API key supplied via Settings to answer free-form questions (rewrite a section, draft an ESG clause). Key stays on device.

**Build / deploy.**
- No build step — `index.html` is the whole thing.
- Deploy: push to GitHub, enable Pages (source = legacy, branch = main, path = /). `.nojekyll` is in place. Live in 60-180 s.

**Constants.**
- `TOOLNAME` = `'fallpaper'`
- `VERSION` = `'1.0.0'`
- `PRIME` = `733` (IFA suite window 719-739)
- `SCHEMA_VERSION` = `'1.0'`

**Verify protocol.** Open the live URL, then in the console:

```javascript
FALLPAPER.version       // '1.0.0'
FALLPAPER.prime         // 733
FALLPAPER.state.templates.length   // 8
FALLPAPER.T0_RULES.length          // 12
FALLPAPER.renderTemplate('suitability', FALLPAPER.state.clients[0].id)
```

Switch tabs (Dashboard → Clients → Generate → Library → Templates → Firm → Audit → Q&A). The demo data should render Marcus Osei and a draft suitability report. Edit the firm, click save, watch the broadcast fire on the `fall-client` channel.

**14-pt gate compliance.** Single HTML file · <400 KB (~130 KB) · domain-appropriate views · multi-template engine · T0 offline · routing by selection (no LLM required) · IDB primary · mobile-first · helpful empty state (demo client) · KONOMI shim · fall-client + fall-signal mesh · PWA manifest baked · two-audience README · MIT.

---

## License

MIT — see `LICENSE`.

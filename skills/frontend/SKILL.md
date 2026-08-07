# Frontend — Skills Consolidadas

Skills reunidas: **frontend-api-integration-patterns**, **frontend-lighthouse**, **redesign-existing-projects**, **review-animations** e **design-it**. Cobrem integração frontend-API, auditoria de performance (Lighthouse/Core Web Vitals), redesign de UI existente, revisão de animações e roteamento para estilos visuais.

## Índice

- [frontend-api-integration-patterns](#frontend-api-integration-patterns)
- [frontend-lighthouse](#frontend-lighthouse)
- [redesign-existing-projects](#redesign-existing-projects)
- [review-animations](#review-animations)
- [design-it](#design-it)

---

## frontend-api-integration-patterns

> **Descrição original:** Production-ready patterns for integrating frontend applications with backend APIs, including race condition handling, request cancellation, retry strategies, error normalization, and UI state management.
>
> **Fonte:** `community`  
> **Autor:** avij1109

### Overview

This skill provides production-ready patterns for integrating frontend applications with backend APIs.

Most frontend issues are not caused by APIs being difficult to call, but by **incorrect handling of asynchronous behavior**—leading to race conditions, stale data, duplicated requests, and poor user experience.

This skill focuses on **correctness, resilience, and user experience**, not just making API calls work.

---

### When to Use This Skill

* Connecting frontend apps (React, React Native, Vue, etc.) to backend APIs
* Integrating ML/AI endpoints (`/predict`, `/recommend`)
* Handling asynchronous data in UI
* Fixing stale data, flickering UI, or duplicate requests
* Designing scalable frontend API layers

---

### Core Patterns

#### 1. API Layer (Separation of Concerns)

Centralize API logic and normalize errors.

```js id="k1m7r2"
export class ApiError extends Error {
  constructor(message, status, payload = null) {
    super(message);
    this.name = "ApiError";
    this.status = status;
    this.payload = payload;
  }
}

export const apiClient = async (url, options = {}) => {
  const res = await fetch(url, {
    headers: { "Content-Type": "application/json" },
    ...options,
  });

  if (!res.ok) {
    let payload = null;
    try {
      payload = await res.json();
    } catch (_) {}

    throw new ApiError(
      payload?.message || "Request failed",
      res.status,
      payload
    );
  }

  // handle empty responses safely (e.g. 204 No Content)
  if (res.status === 204) return null;

  const text = await res.text();
  return text ? JSON.parse(text) : null;
};
```

---

#### 2. Race-Safe State Management

Prevent stale responses from overwriting fresh data.

```js id="y7p4ha"
useEffect(() => {
  let cancelled = false;

  const load = async () => {
    try {
      setLoading(true);
      setError(null);

      const result = await getUser();

      if (!cancelled) setData(result);
    } catch (err) {
      if (!cancelled) setError(err.message);
    } finally {
      if (!cancelled) setLoading(false);
    }
  };

  load();

  return () => {
    cancelled = true;
  };
}, []);
```

> Use a cancellation flag for non-fetch async logic. For network requests, prefer AbortController.

---

#### 3. Request Cancellation (AbortController)

Cancel in-flight requests to avoid memory leaks and stale updates.

```js id="l9x2pw"
useEffect(() => {
  const controller = new AbortController();

  const load = async () => {
    try {
      const data = await getUser({ signal: controller.signal });
      setData(data);
    } catch (err) {
      if (err.name === "AbortError") return;
      setError(err.message);
    }
  };

  load();
  return () => controller.abort();
}, [userId]);
```

---

#### 4. Retry with Exponential Backoff

Retry only transient failures (5xx or network errors).

```js id="8n3zcf"
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

const fetchWithBackoff = async (fn, retries = 3, delay = 300) => {
  try {
    return await fn();
  } catch (err) {
    const isAbort = err.name === "AbortError";
    const isHttpError = typeof err.status === "number";
    const isRetryable = !isAbort && (!isHttpError || err.status >= 500);

    if (retries <= 0 || !isRetryable) throw err;

    const nextDelay = delay * 2 + Math.random() * 100;
    await sleep(nextDelay);

    return fetchWithBackoff(fn, retries - 1, nextDelay);
  }
};
```

---

#### 5. Debounced API Calls

Avoid excessive API calls (e.g., search inputs).

```js id="i2r7wq"
const useDebounce = (value, delay = 400) => {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(t);
  }, [value, delay]);

  return debounced;
};
```

---

#### 6. Request Deduplication

Prevent duplicate API calls across components.

```js id="x8v4km"
const inFlight = new Map();

export const dedupedFetch = (key, fn) => {
  if (inFlight.has(key)) return inFlight.get(key);

  const promise = fn().finally(() => inFlight.delete(key));
  inFlight.set(key, promise);
  return promise;
};
```

---

### Examples

#### Example 1: ML Prediction with Cancellation

```js id="n5q2pt"
const controllerRef = useRef(null);

const handlePredict = async (input) => {
  controllerRef.current?.abort();
  controllerRef.current = new AbortController();

  try {
    const result = await fetchWithBackoff(() =>
      apiClient("/predict", {
        method: "POST",
        body: JSON.stringify({ text: input }),
        signal: controllerRef.current.signal,
      })
    );

    setOutput(result);
  } catch (err) {
    if (err.name === "AbortError") return;
    setError(err.message);
  }
};
```

---

#### Example 2: Debounced Search

```js id="w4z8yn"
const debouncedQuery = useDebounce(query, 400);

useEffect(() => {
  if (!debouncedQuery) return;

  const controller = new AbortController();

  searchAPI(debouncedQuery, { signal: controller.signal })
    .then(setResults)
    .catch((err) => {
      if (err.name !== "AbortError") {
        setError("Search failed. Please try again.");
      }
    });

  return () => controller.abort();
}, [debouncedQuery]);
```

---

#### Example 3: Optimistic UI Update

```js id="q2k9hz"
const deleteItem = async (id) => {
  const previous = items;

  setItems((curr) => curr.filter((item) => item.id !== id));

  try {
    await apiClient(`/items/${id}`, { method: "DELETE" });
  } catch (err) {
    setItems(previous);
    setError("Delete failed. Please try again.");
  }
};
```

---

### Best Practices

* ✅ Centralize API logic in a dedicated layer
* ✅ Normalize errors using a custom error class
* ✅ Always handle loading, error, and success states
* ✅ Use AbortController for request cancellation
* ✅ Retry only transient failures (5xx)
* ✅ Use debouncing for input-driven APIs
* ✅ Deduplicate identical requests

---

### Anti-Patterns

* ❌ Retrying 4xx errors
* ❌ No request cancellation (memory leaks)
* ❌ Race-condition-prone state updates
* ❌ Swallowing errors silently
* ❌ Global loading/error state for multiple requests
* ❌ Calling APIs directly inside components repeatedly

---

### Common Pitfalls

**Problem:** UI shows stale data
**Solution:** Use cancellation or guard against outdated responses

**Problem:** Too many API calls on input
**Solution:** Use debouncing + cancellation

**Problem:** Duplicate requests from multiple components
**Solution:** Use request deduplication

**Problem:** Server overload during retry
**Solution:** Use exponential backoff

**Problem:** State updates after component unmount
**Solution:** Use AbortController cleanup

---

### Limitations

* These examples use vanilla JavaScript patterns; adapt them to your framework's data-fetching library when using React Query, SWR, Apollo, Relay, or similar tools.
* Do not retry non-idempotent mutations unless the backend provides idempotency keys or another duplicate-safe contract.
* Do not expose privileged API keys in frontend code; proxy sensitive requests through a backend.

---

### Additional Resources

* https://developer.mozilla.org/en-US/docs/Web/API/AbortController
* https://react.dev
* https://axios-http.com

---

---

## frontend-lighthouse

> **Descrição original:** Add a portable Lighthouse CI gate for production frontend builds with Core Web Vitals budgets, category floors, median runs, and CI artifacts.
>
> **Fonte:** `stareezy-1/frontend-architecture-skill`  
> **Autor:** stareezy-1

> Portable skill — readable by Claude Code, OpenCode, Codex, Cursor, Windsurf, and others.
> This skill describes a **CI performance gate** — a Lighthouse CI config plus a workflow — not a
> component library or a visual style. It pairs with the **frontend-seo** and
> **frontend-architecture** skills: SEO writes the metadata, Lighthouse proves it ships fast.

The goal: every pull request is **blocked unless the production build meets explicit Core Web
Vitals budgets and category score floors**. Budgets live in **one** `lighthouserc.cjs`, runs are
**median-of-N** so the gate doesn't flake, and the same config runs locally and in CI.

### When to Use This Skill

- Use when adding a Lighthouse CI performance gate to a web app.
- Use when setting Core Web Vitals budgets for LCP, CLS, and TBT as the lab proxy for INP.
- Use when configuring category score floors for performance, SEO, accessibility, and best practices.
- Use when debugging flaky Lighthouse runs or making reports visible as CI artifacts.

---

### 0. The five core ideas

1. **One config, one source of truth.** All budgets and assertions live in a single `lighthouserc.cjs`. Named constants for each budget — no magic numbers buried in assertion objects.
2. **Gate the production build, never dev.** Lighthouse runs against `build` + `start` (the real, optimized output). Dev-server numbers are meaningless for a budget.
3. **Median-of-N kills flakiness.** Run 3+ times and assert on the median run, so per-run jitter (cold caches, CI noise) never red-flags a healthy build.
4. **Budgets encode Google's "good" thresholds.** LCP ≤ 2500 ms, INP ≤ 200 ms (gated via the TBT lab proxy), CLS ≤ 0.1 — the values that earn green scores, not "needs improvement".
5. **Blocking in CI, visible as artifacts.** A GitHub Action runs the gate on every PR touching the app and uploads the HTML/JSON reports so failures are debuggable.

---

### 1. Files this skill adds

```
apps/web/                          (or your app root)
├── lighthouserc.cjs               ← the gate: budgets + assertions + collect settings
├── package.json                   ← "lhci": "lhci autorun --config=./lighthouserc.cjs"
└── .github/workflows/lighthouse.yml  ← PR-blocking CI job (build → start → lhci → upload)
```

Plus a dev dependency: `@lhci/cli`.

```bash
pnpm add -D @lhci/cli        # or npm i -D / yarn add -D
```

---

### 2. The config (`lighthouserc.cjs`)

`.cjs` (CommonJS) so it loads without ESM/TS transpilation. Every budget is a **named constant**
with a comment explaining the threshold — never a bare number inside an assertion.

```js
/**
 * Lighthouse CI configuration — Core Web Vitals budgets for the marketing surface.
 *
 * Enforces Google's mobile "good" CWV thresholds:
 *   - Largest Contentful Paint (LCP) ≤ 2500 ms
 *   - Cumulative Layout Shift (CLS)  ≤ 0.1
 *   - Interaction to Next Paint (INP) ≤ 200 ms
 *
 * INP is a *field* metric with no direct lab audit, so in the lab we gate on
 * Total Blocking Time (TBT) — Lighthouse's recommended lab proxy — at the same
 * budget, and assert the experimental INP audit directly as a warning where the
 * build exposes it.
 *
 * Collection runs against the *production* server (build + start) on Lighthouse's
 * default mobile (Moto G4 / slow 4G) emulation.
 */

/** The fixed port the production server is started on for the audit. */
const PORT = 3100;
const BASE_URL = `http://localhost:${PORT}`;

/** Pages whose budgets are enforced in CI. */
const MARKETING_URLS = [`${BASE_URL}/`];

/**
 * Core Web Vitals budgets on mobile — Google's "good" thresholds.
 * These are the values that earn the best Lighthouse scores.
 */
const LCP_BUDGET_MS = 2500; // good
const INP_BUDGET_MS = 200; // good (TBT lab proxy)
const CLS_BUDGET = 0.1; // good

module.exports = {
  ci: {
    collect: {
      // Build is run separately in CI; here we only serve the production output.
      startServerCommand: `pnpm start --port ${PORT}`,
      startServerReadyPattern: "Ready in", // framework's "server ready" log line
      startServerReadyTimeout: 120000,
      url: MARKETING_URLS,
      // Median of multiple runs keeps the gate stable against per-run jitter.
      numberOfRuns: 3,
      settings: {
        // Default mobile emulation; opt into desktop via env for a second run.
        preset:
          process.env.LHCI_FORM_FACTOR === "desktop" ? "desktop" : undefined,
        // Only gate the categories we care about; skip PWA category noise.
        onlyCategories: [
          "performance",
          "seo",
          "accessibility",
          "best-practices",
        ],
      },
    },
    assert: {
      // Median across runs is the value compared against each budget.
      aggregationMethod: "median-run",
      assertions: {
        // --- Core Web Vitals budgets (the contract) ---------------------
        "largest-contentful-paint": [
          "error",
          { maxNumericValue: LCP_BUDGET_MS },
        ],
        "cumulative-layout-shift": ["error", { maxNumericValue: CLS_BUDGET }],
        "total-blocking-time": ["error", { maxNumericValue: INP_BUDGET_MS }],
        // Direct INP audit where the Lighthouse build exposes it (else ignored).
        "interaction-to-next-paint": [
          "warn",
          { maxNumericValue: INP_BUDGET_MS },
        ],

        // --- Category floors (target top Lighthouse scores) -------------
        "categories:performance": ["error", { minScore: 0.9 }],
        "categories:seo": ["error", { minScore: 0.95 }],
        "categories:accessibility": ["error", { minScore: 0.95 }],
        "categories:best-practices": ["error", { minScore: 0.9 }],
      },
    },
    upload: {
      // Keep reports in the CI run's filesystem; no external LHCI server.
      target: "filesystem",
      outputDir: "./.lighthouseci",
    },
  },
};
```

**Hard rules:**

- Every budget is a named constant with a unit in its name (`LCP_BUDGET_MS`) and a comment.
- `aggregationMethod: "median-run"` is non-negotiable — single-run gates flake constantly.
- `numberOfRuns` ≥ 3 (odd numbers give a clean median).
- Assert on TBT for INP in the lab; treat the experimental `interaction-to-next-paint` audit as a `warn`, not an `error` (it isn't present in every Lighthouse build).
- Keep `onlyCategories` to exactly what you gate — fewer audits, faster, less noise.

---

### 3. Choosing budget severity and thresholds

| Audit / category            | Severity | Threshold | Why                                                   |
| --------------------------- | -------- | --------- | ----------------------------------------------------- |
| `largest-contentful-paint`  | `error`  | ≤ 2500 ms | Google "good" LCP                                     |
| `cumulative-layout-shift`   | `error`  | ≤ 0.1     | Google "good" CLS                                     |
| `total-blocking-time`       | `error`  | ≤ 200 ms  | INP lab proxy                                         |
| `interaction-to-next-paint` | `warn`   | ≤ 200 ms  | not in all builds; don't hard-fail on a missing audit |
| `categories:performance`    | `error`  | ≥ 0.9     | top (green) band                                      |
| `categories:seo`            | `error`  | ≥ 0.95    | SEO is cheap to keep perfect                          |
| `categories:accessibility`  | `error`  | ≥ 0.95    | a11y regressions must block                           |
| `categories:best-practices` | `error`  | ≥ 0.9     | green band                                            |

Use `error` for contracts that must hold and `warn` for audits that are environment-dependent or
aspirational. **Start strict and only loosen with a recorded reason** — a budget you keep raising
to make CI pass is a budget that no longer protects anything.

---

### 4. The npm script

```jsonc
// package.json
{
  "scripts": {
    "lhci": "lhci autorun --config=./lighthouserc.cjs"
  }
}
```

`lhci autorun` runs `collect` → `assert` → `upload` in sequence. Run it locally before pushing to
reproduce exactly what CI does:

```bash
pnpm build && pnpm lhci
# desktop form factor:
LHCI_FORM_FACTOR=desktop pnpm build && LHCI_FORM_FACTOR=desktop pnpm lhci
```

---

### 5. The GitHub Actions workflow

Runs on PRs that touch the app or the workflow itself. Builds the production output, runs the
gate, and **always** uploads the reports (even on failure) so a red check is debuggable.

```yaml
name: Lighthouse CWV

on:
  pull_request:
    branches: [main]
    paths:
      - "apps/web/**"
      - ".github/workflows/lighthouse.yml"

permissions:
  contents: read

jobs:
  lighthouse:
    name: Lighthouse CWV (marketing pages)
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: apps/web
    steps:
      - uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4 # version comes from root package.json packageManager

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm

      - name: Install dependencies
        working-directory: .
        run: pnpm install --frozen-lockfile

      - name: Build web app
        run: pnpm build

      # build + start the production server, run Lighthouse on mobile emulation,
      # fail the job if any budget in lighthouserc.cjs is exceeded.
      - name: Run Lighthouse CI
        run: pnpm lhci

      - name: Upload Lighthouse reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: lighthouse-reports
          path: apps/web/.lighthouseci
          if-no-files-found: ignore
```

**Hard rules:**

- Trigger on the app path **and** the workflow file so config changes are self-testing.
- `if: always()` on the upload step — you need the report most when the gate fails.
- Gate on the **production** build (`pnpm build` then the `start` server in `collect`).
- Match the CI Node/pnpm versions to the repo's pinned versions to avoid lockfile drift.

---

### 6. Framework adapters

The config is framework-neutral except `startServerCommand` and `startServerReadyPattern`.

| Framework     | `startServerCommand`                                              | `startServerReadyPattern`                   |
| ------------- | ----------------------------------------------------------------- | ------------------------------------------- |
| **Next.js**   | `pnpm start --port 3100` (after `next build`)                     | `"Ready in"`                                |
| **Remix**     | `pnpm start` (serve the built app)                                | server's listening log line                 |
| **Astro**     | `node ./dist/server/entry.mjs` (SSR) or `npx serve dist` (static) | the adapter's ready line / serve's URL line |
| **SvelteKit** | `node build` (node adapter)                                       | `"Listening on"`                            |
| **Vite SPA**  | `npx vite preview --port 3100`                                    | `"Local:"`                                  |

For purely static output you can skip the server and point `collect.staticDistDir` at the build
folder instead of `startServerCommand` — Lighthouse serves it internally.

---

### 7. Debugging failing or flaky runs

- **Flaky LCP/TBT** → raise `numberOfRuns` (5), confirm `median-run`, and make sure nothing else is competing for CPU on the runner.
- **`interaction-to-next-paint` errors** → it should be `warn`, not `error`; the audit is missing in some Lighthouse versions.
- **"server not ready" timeout** → fix `startServerReadyPattern` to match the framework's actual ready log, and raise `startServerReadyTimeout`.
- **Real regressions** → open the uploaded report artifact, read the failed audit's "Opportunities"/"Diagnostics", fix the cause (oversized image, render-blocking JS, layout shift from unsized media) — don't just bump the budget.
- **Desktop vs mobile divergence** → run both form factors; mobile is the stricter gate and should be the default.

---

### 8. Conventions checklist (enforce in review)

- [ ] All budgets are named constants with units and comments — no magic numbers in assertions.
- [ ] Gate runs against the **production** build, never the dev server.
- [ ] `aggregationMethod: "median-run"` with `numberOfRuns` ≥ 3.
- [ ] CWV budgets at Google "good" thresholds (LCP ≤ 2500, TBT ≤ 200, CLS ≤ 0.1).
- [ ] INP gated via TBT (`error`); experimental INP audit is `warn`.
- [ ] Category floors set as `error` (perf ≥ 0.9, SEO/a11y ≥ 0.95, best-practices ≥ 0.9).
- [ ] `onlyCategories` lists exactly the gated categories.
- [ ] CI triggers on the app path **and** the workflow file; reports upload with `if: always()`.
- [ ] Local `pnpm lhci` reproduces the CI run.
- [ ] Budgets are tightened over time, loosened only with a recorded reason.

---

### 9. How to apply this skill

**Adding the gate to a project:** install `@lhci/cli`, drop in `lighthouserc.cjs` with your URLs
and `startServerCommand`, add the `lhci` script, and add the workflow. Run `pnpm build && pnpm lhci`
locally to confirm it passes before opening a PR.

**Adding a page to the gate:** append its URL to `MARKETING_URLS` (or a second URL array). Each URL
is audited independently against the same budgets.

**Tuning budgets:** change the named constant, not the assertion. Record why in the comment. Prefer
fixing the regression over raising the budget.

**Reviewing performance:** run the checklist in §8. The highest-value catches are a gate that runs
against the dev server (meaningless numbers) and single-run assertions (chronic flakiness).

---

### Publishing / installing this skill

This skill follows the Anthropic `SKILL.md` format and is portable across agents.

1. Keep it under `skills/frontend-lighthouse/SKILL.md` in a public GitHub repo.
2. Keep the frontmatter `name` and high-signal `description` — discovery indexes match against it.
3. Install with: `npx skills add <org>/<repo> --skill "frontend-lighthouse"`.
4. Non-`SKILL.md` agents can be pointed here from `AGENTS.md` / `CLAUDE.md`; Kiro can mirror it as a steering file.

### Limitations

- Lighthouse CI is a lab signal and does not replace field monitoring from real-user metrics.
- Budgets must be tuned to the actual app route, hosting platform, and device/network assumptions.
- A passing Lighthouse gate does not prove business-critical flows, visual correctness, or backend availability.

---

## redesign-existing-projects

> **Descrição original:** Use when upgrading existing websites or apps by auditing generic UI patterns and applying premium design fixes without rewrites.
>
> **Fonte:** `Leonxlnx/taste-skill`  
> **Autor:** Leonxlnx

### When to Use

- Use when the user asks to redesign, restyle, modernize, polish, or improve an existing website or app UI.
- Use when the task is to audit current frontend code and make targeted visual improvements without changing the product architecture.
- Use when the design feels generic, AI-generated, poorly spaced, visually flat, or missing responsive, interactive, loading, empty, or error states.

### Limitations

- This skill upgrades existing UI but does not authorize framework migrations, information-architecture rewrites, or product-scope expansion by default.
- Preserve working behavior, routing, data flows, accessibility semantics, and tests while making visual changes.
- Validate redesigned screens in the actual app across supported browsers and viewport sizes before considering the work complete.


### How This Works

When applied to an existing project, follow this sequence:

1. **Scan** — Read the codebase. Identify the framework, styling method (Tailwind, vanilla CSS, styled-components, etc.), and current design patterns.
2. **Diagnose** — Run through the audit below. List every generic pattern, weak point, and missing state you find.
3. **Fix** — Apply targeted upgrades working with the existing stack. Do not rewrite from scratch. Improve what's there.

### Design Audit

#### Typography

Check for these problems and fix them:

- **Browser default fonts or Inter everywhere.** Replace with a font that has character. Good options: `Geist`, `Outfit`, `Cabinet Grotesk`, `Satoshi`. For editorial/creative projects, pair a serif header with a sans-serif body.
- **Headlines lack presence.** Increase size for display text, tighten letter-spacing, reduce line-height. Headlines should feel heavy and intentional.
- **Body text too wide.** Limit paragraph width to roughly 65 characters. Increase line-height for readability.
- **Only Regular (400) and Bold (700) weights used.** Introduce Medium (500) and SemiBold (600) for more subtle hierarchy.
- **Numbers in proportional font.** Use a monospace font or enable tabular figures (`font-variant-numeric: tabular-nums`) for data-heavy interfaces.
- **Missing letter-spacing adjustments.** Use negative tracking for large headers, positive tracking for small caps or labels.
- **All-caps subheaders everywhere.** Try lowercase italics, sentence case, or small-caps instead.
- **Orphaned words.** Single words sitting alone on the last line. Fix with `text-wrap: balance` or `text-wrap: pretty`.

#### Color and Surfaces

- **Pure `#000000` background.** Replace with off-black, dark charcoal, or tinted dark (`#0a0a0a`, `#121212`, or a dark navy).
- **Oversaturated accent colors.** Keep saturation below 80%. Desaturate accents so they blend with neutrals instead of screaming.
- **More than one accent color.** Pick one. Remove the rest. Consistency beats variety.
- **Mixing warm and cool grays.** Stick to one gray family. Tint all grays with a consistent hue (warm or cool, not both).
- **Purple/blue "AI gradient" aesthetic.** This is the most common AI design fingerprint. Replace with neutral bases and a single, considered accent.
- **Generic `box-shadow`.** Tint shadows to match the background hue. Use colored shadows (e.g., dark blue shadow on a blue background) instead of pure black at low opacity.
- **Flat design with zero texture.** Add subtle noise, grain, or micro-patterns to backgrounds. Pure flat vectors feel sterile.
- **Perfectly even gradients.** Break the uniformity with radial gradients, noise overlays, or mesh gradients instead of standard linear 45-degree fades.
- **Inconsistent lighting direction.** Audit all shadows to ensure they suggest a single, consistent light source.
- **Random dark sections in a light mode page (or vice versa).** A single dark-background section breaking an otherwise light page looks like a copy-paste accident. Either commit to a full dark mode or keep a consistent background tone throughout. If contrast is needed, use a slightly darker shade of the same palette — not a sudden jump to `#111` in the middle of a cream page.
- **Empty, flat sections with no visual depth.** Sections that are just text on a plain background feel unfinished. Add high-quality background imagery (blurred, overlaid, or masked), subtle patterns, or ambient gradients. Use reliable placeholder sources like `https://picsum.photos/seed/{name}/1920/1080` when real assets are not available. Experiment with background images behind hero sections, feature blocks, or CTAs — even a subtle full-width photo at low opacity adds presence.

#### Layout

- **Everything centered and symmetrical.** Break symmetry with offset margins, mixed aspect ratios, or left-aligned headers over centered content.
- **Three equal card columns as feature row.** This is the most generic AI layout. Replace with a 2-column zig-zag, asymmetric grid, horizontal scroll, or masonry layout.
- **Using `height: 100vh` for full-screen sections.** Replace with `min-height: 100dvh` to prevent layout jumping on mobile browsers (iOS Safari viewport bug).
- **Complex flexbox percentage math.** Replace with CSS Grid for reliable multi-column structures.
- **No max-width container.** Add a container constraint (around 1200-1440px) with auto margins so content doesn't stretch edge-to-edge on wide screens.
- **Cards of equal height forced by flexbox.** Allow variable heights or use masonry when content varies in length.
- **Uniform border-radius on everything.** Vary the radius: tighter on inner elements, softer on containers.
- **No overlap or depth.** Elements sit flat next to each other. Use negative margins to create layering and visual depth.
- **Symmetrical vertical padding.** Top and bottom padding are always identical. Adjust optically — bottom padding often needs to be slightly larger.
- **Dashboard always has a left sidebar.** Try top navigation, a floating command menu, or a collapsible panel instead.
- **Missing whitespace.** Double the spacing. Let the design breathe. Dense layouts work for data dashboards, not for marketing pages.
- **Buttons not bottom-aligned in card groups.** When cards have different content lengths, CTAs end up at random heights. Pin buttons to the bottom of each card so they form a clean horizontal line regardless of content above.
- **Feature lists starting at different vertical positions.** In pricing tables or comparison cards, the list of features should start at the same Y position across all columns. Use consistent spacing above the list or fixed-height title/price blocks.
- **Inconsistent vertical rhythm in side-by-side elements.** When placing cards, columns, or panels next to each other, align shared elements (titles, descriptions, prices, buttons) across all items. Misaligned baselines make the layout look broken.
- **Mathematical alignment that looks optically wrong.** Centering by the math doesn't always look centered to the eye. Icons next to text, play buttons in circles, or text in buttons often need 1-2px optical adjustments to feel right.

#### Interactivity and States

- **No hover states on buttons.** Add background shift, slight scale, or translate on hover.
- **No active/pressed feedback.** Add a subtle `scale(0.98)` or `translateY(1px)` on press to simulate a physical click.
- **Instant transitions with zero duration.** Add smooth transitions (200-300ms) to all interactive elements.
- **Missing focus ring.** Ensure visible focus indicators for keyboard navigation. This is an accessibility requirement, not optional.
- **No loading states.** Replace generic circular spinners with skeleton loaders that match the layout shape.
- **No empty states.** An empty dashboard showing nothing is a missed opportunity. Design a composed "getting started" view.
- **No error states.** Add clear, inline error messages for forms. Do not use `window.alert()`.
- **Dead links.** Buttons that link to `#`. Either link to real destinations or visually disable them.
- **No indication of current page in navigation.** Style the active nav link differently so users know where they are.
- **Scroll jumping.** Anchor clicks jump instantly. Add `scroll-behavior: smooth`.
- **Animations using `top`, `left`, `width`, `height`.** Switch to `transform` and `opacity` for GPU-accelerated, smooth animation.

#### Content

- **Generic names like "John Doe" or "Jane Smith".** Use diverse, realistic-sounding names.
- **Fake round numbers like `99.99%`, `50%`, `$100.00`.** Use organic, messy data: `47.2%`, `$99.00`, `+1 (312) 847-1928`.
- **Placeholder company names like "Acme Corp", "Nexus", "SmartFlow".** Invent contextual, believable brand names.
- **AI copywriting cliches.** Never use "Elevate", "Seamless", "Unleash", "Next-Gen", "Game-changer", "Delve", "Tapestry", or "In the world of...". Write plain, specific language.
- **Exclamation marks in success messages.** Remove them. Be confident, not loud.
- **"Oops!" error messages.** Be direct: "Connection failed. Please try again."
- **Passive voice.** Use active voice: "We couldn't save your changes" instead of "Mistakes were made."
- **All blog post dates identical.** Randomize dates to appear real.
- **Same avatar image for multiple users.** Use unique assets for every distinct person.
- **Lorem Ipsum.** Never use placeholder latin text. Write real draft copy.
- **Title Case On Every Header.** Use sentence case instead.

#### Component Patterns

- **Generic card look (border + shadow + white background).** Remove the border, or use only background color, or use only spacing. Cards should exist only when elevation communicates hierarchy.
- **Always one filled button + one ghost button.** Add text links or tertiary styles to reduce visual noise.
- **Pill-shaped "New" and "Beta" badges.** Try square badges, flags, or plain text labels.
- **Accordion FAQ sections.** Use a side-by-side list, searchable help, or inline progressive disclosure.
- **3-card carousel testimonials with dots.** Replace with a masonry wall, embedded social posts, or a single rotating quote.
- **Pricing table with 3 towers.** Highlight the recommended tier with color and emphasis, not just extra height.
- **Modals for everything.** Use inline editing, slide-over panels, or expandable sections instead of popups for simple actions.
- **Avatar circles exclusively.** Try squircles or rounded squares for a less generic look.
- **Light/dark toggle always a sun/moon switch.** Use a dropdown, system preference detection, or integrate it into settings.
- **Footer link farm with 4 columns.** Simplify. Focus on main navigational paths and legally required links.

#### Iconography

- **Lucide or Feather icons exclusively.** These are the "default" AI icon choice. Use Phosphor, Heroicons, or a custom set for differentiation.
- **Rocketship for "Launch", shield for "Security".** Replace cliche metaphors with less obvious icons (bolt, fingerprint, spark, vault).
- **Inconsistent stroke widths across icons.** Audit all icons and standardize to one stroke weight.
- **Missing favicon.** Always include a branded favicon.
- **Stock "diverse team" photos.** Use real team photos, candid shots, or a consistent illustration style instead of uncanny stock imagery.

#### Code Quality

- **Div soup.** Use semantic HTML: `<nav>`, `<main>`, `<article>`, `<aside>`, `<section>`.
- **Inline styles mixed with CSS classes.** Move all styling to the project's styling system.
- **Hardcoded pixel widths.** Use relative units (`%`, `rem`, `em`, `max-width`) for flexible layouts.
- **Missing alt text on images.** Describe image content for screen readers. Never leave `alt=""` or `alt="image"` on meaningful images.
- **Arbitrary z-index values like `9999`.** Establish a clean z-index scale in the theme/variables.
- **Commented-out dead code.** Remove all debug artifacts before shipping.
- **Import hallucinations.** Check that every import actually exists in `package.json` or the project dependencies.
- **Missing meta tags.** Add proper `<title>`, `description`, `og:image`, and social sharing meta tags.

#### Strategic Omissions (What AI Typically Forgets)

- **No legal links.** Add privacy policy and terms of service links in the footer.
- **No "back" navigation.** Dead ends in user flows. Every page needs a way back.
- **No custom 404 page.** Design a helpful, branded "page not found" experience.
- **No form validation.** Add client-side validation for emails, required fields, and format checks.
- **No "skip to content" link.** Essential for keyboard users. Add a hidden skip-link.
- **No cookie consent.** If required by jurisdiction, add a compliant consent banner.

### Upgrade Techniques

When upgrading a project, pull from these high-impact techniques to replace generic patterns:

#### Typography Upgrades
- **Variable font animation.** Interpolate weight or width on scroll or hover for text that feels alive.
- **Outlined-to-fill transitions.** Text starts as a stroke outline and fills with color on scroll entry or interaction.
- **Text mask reveals.** Large typography acting as a window to video or animated imagery behind it.

#### Layout Upgrades
- **Broken grid / asymmetry.** Elements that deliberately ignore column structure — overlapping, bleeding off-screen, or offset with calculated randomness.
- **Whitespace maximization.** Aggressive use of negative space to force focus on a single element.
- **Parallax card stacks.** Sections that stick and physically stack over each other during scroll.
- **Split-screen scroll.** Two halves of the screen sliding in opposite directions.

#### Motion Upgrades
- **Smooth scroll with inertia.** Decouple scrolling from browser defaults for a heavier, cinematic feel.
- **Staggered entry.** Elements cascade in with slight delays, combining Y-axis translation with opacity fade. Never mount everything at once.
- **Spring physics.** Replace linear easing with spring-based motion for a natural, weighty feel on all interactive elements.
- **Scroll-driven reveals.** Content entering through expanding masks, wipes, or draw-on SVG paths tied to scroll progress.

#### Surface Upgrades
- **True glassmorphism.** Go beyond `backdrop-filter: blur`. Add a 1px inner border and a subtle inner shadow to simulate edge refraction.
- **Spotlight borders.** Card borders that illuminate dynamically under the cursor.
- **Grain and noise overlays.** A fixed, pointer-events-none overlay with subtle noise to break digital flatness.
- **Colored, tinted shadows.** Shadows that carry the hue of the background rather than using generic black.

### Fix Priority

Apply changes in this order for maximum visual impact with minimum risk:

1. **Font swap** — biggest instant improvement, lowest risk
2. **Color palette cleanup** — remove clashing or oversaturated colors
3. **Hover and active states** — makes the interface feel alive
4. **Layout and spacing** — proper grid, max-width, consistent padding
5. **Replace generic components** — swap cliche patterns for modern alternatives
6. **Add loading, empty, and error states** — makes it feel finished
7. **Polish typography scale and spacing** — the premium final touch

### Rules

- Work with the existing tech stack. Do not migrate frameworks or styling libraries.
- Do not break existing functionality. Test after every change.
- Before importing any new library, check the project's dependency file first.
- If the project uses Tailwind, check the version (v3 vs v4) before modifying config.
- If the project has no framework, use vanilla CSS.
- Keep changes reviewable and focused. Small, targeted improvements over big rewrites.

---

## review-animations

> **Descrição original:** Use when reviewing animation and motion code against a strict craft, performance, accessibility, and interaction-quality bar.
>
> **Fonte:** `emilkowalski/skills`  
> **Autor:** Emil Kowalski

### When to Use

- Use when the user asks for an animation, motion, or interaction review.
- Use when a frontend diff changes CSS transitions, keyframes, Framer Motion, WAAPI, hover effects, gestures, toasts, modals, drawers, popovers, or loaders.
- Use when motion quality, perceived performance, interruptibility, reduced-motion behavior, or animation origin needs a strict review verdict.

### Limitations

- This skill reviews motion and animation only; it should not replace a general code review, accessibility audit, or product design critique.
- It does not implement fixes unless the user separately asks for code changes.
- Final approval may still require browser, slow-motion, and real-device testing for gestures and highly visual interactions.

### Examples

Ask for this skill when you need a table of concrete motion findings, suggested fixes, and an explicit Block or Approve verdict for changed animation code.

A specialized review skill. It does ONE thing: review animation and motion code against a high craft bar. It does not write features, fix unrelated bugs, or review non-motion code. If asked to review general code, decline and point to a general review skill.

### Operating Posture

You are a senior motion-design reviewer with a brutal eye for craft. Your bias is toward **motion that feels right**, not motion that merely runs. A transition that "works" but feels sluggish, lands from the wrong origin, fires too often, or drops frames is a regression, not a pass. Default to flagging. Approval is earned, not assumed.

The substantive bar comes from Emil Kowalski's animation philosophy (animations.dev). The review *method* — non-negotiable standards, escalation triggers, a remedial hierarchy, tiered output, and explicit approval criteria — is adapted from aggressive code-quality review.

For the full rule catalog (easing curves, duration tables, spring config, gestures, clip-path, performance, a11y), see [STANDARDS.md](STANDARDS.md). Load it whenever a finding needs a precise value or citation.

### The Ten Non-Negotiable Standards

Every animation in the diff is measured against these. A violation is a finding.

1. **Justified motion.** Every animation must answer "why does this animate?" — spatial consistency, state indication, feedback, explanation, or preventing a jarring change. "It looks cool" on a frequently-seen element is a block.

2. **Frequency-appropriate.** Match motion to how often it's seen. Keyboard-initiated and 100+/day actions get **no** animation. Tens/day gets reduced motion. Occasional gets standard. Rare/first-time can have delight.

3. **Responsive easing.** Entering/exiting elements use `ease-out` or a strong custom curve. `ease-in` on UI is a block — it delays the moment the user watches most. Built-in CSS easings are too weak; expect custom cubic-beziers.

4. **Sub-300ms UI.** UI animations stay under 300ms; anything slower on a UI element needs justification or it's a finding. Per-element budgets live in [STANDARDS.md](STANDARDS.md).

5. **Origin & physical correctness.** Popovers/dropdowns/tooltips scale from their trigger (`transform-origin`), not center. Never animate from `scale(0)` — start from `scale(0.9–0.97)` + opacity (Modals are exempt — they stay centered.)

6. **Interruptibility.** Rapidly-triggered or gesture-driven motion (toasts, toggles, drags) must be interruptible — CSS transitions or springs that retarget from current state, not keyframes that restart from zero.

7. **GPU-only properties.** Animate `transform` and `opacity` only. Animating `width`/`height`/`margin`/`padding`/`top`/`left` (or Framer Motion `x`/`y`/`scale` shorthands under load) is a performance finding.

8. **Accessibility.** `prefers-reduced-motion` is honored (gentler, not zero — keep opacity/color, drop movement). Hover animations are gated behind `@media (hover: hover) and (pointer: fine)`.

9. **Asymmetric enter/exit.** Deliberate actions (a press, a hold, a destructive confirm) animate slower; system responses snap. Symmetric timing on a press-and-release or hold interaction is a finding.

10. **Cohesion.** Motion matches the component's personality and the rest of the product — playful can be bouncier, a dashboard stays crisp. Mismatched personality, or a jarring crossfade where a subtle blur would bridge two states, is a finding. When unsure whether motion feels right, the strongest move is often to delete it.

### Aggressive Escalation Triggers

Flag these on sight, hard:

- `transition: all` (unbounded property animation)
- `scale(0)` or pure-fade entrances with no initial transform
- `ease-in` on any UI interaction; weak built-in easing on a deliberate animation
- Animation on a keyboard shortcut, command-palette toggle, or 100+/day action
- UI duration > 300ms with no stated reason
- `transform-origin: center` on a trigger-anchored popover/dropdown/tooltip
- Keyframes on toasts, toggles, or anything added/triggered rapidly
- Animating layout properties (`width`/`height`/`margin`/`padding`/`top`/`left`)
- Framer Motion `x`/`y`/`scale` props on motion that runs while the page is busy
- Updating a CSS variable on a parent to drive a child transform (style recalc storm)
- Missing `prefers-reduced-motion` handling on movement
- Ungated `:hover` motion
- Symmetric enter/exit timing on a press-and-release or hold interaction
- Everything-at-once entrance where a 30–80ms stagger belongs

### Remedial Preference Hierarchy

When proposing fixes, prefer earlier moves over later ones:

1. **Delete the animation** (high-frequency / no purpose / keyboard-triggered).
2. **Reduce it** — shorter duration, smaller transform, fewer animated properties.
3. **Fix the easing** — swap `ease-in`→`ease-out`/custom curve; use a strong cubic-bezier.
4. **Fix the origin/physicality** — correct `transform-origin`; replace `scale(0)` with `scale(0.95)`+opacity.
5. **Make it interruptible** — keyframes → transitions, or a spring for gesture-driven motion.
6. **Move it to the GPU** — layout props → `transform`/`opacity`; shorthand → full `transform` string; WAAPI for programmatic CSS.
7. **Asymmetric timing** — slow the deliberate phase, snap the response.
8. **Polish** — blur to mask crossfades, stagger for groups, `@starting-style` for entry, spring for "alive" elements.
9. **Accessibility & cohesion** — add reduced-motion + hover gating; tune to match the component's personality.

### Required Output Format

Two parts, in this order.

#### Part 1 — Findings table (REQUIRED)

A single markdown table. One row per issue. Never a "Before:/After:" list.

| Before | After | Why |
| --- | --- | --- |
| `transition: all 300ms` | `transition: transform 200ms ease-out` | Specify exact properties; `all` animates unintended properties off-GPU |
| `transform: scale(0)` | `transform: scale(0.95); opacity: 0` | Nothing appears from nothing — `scale(0)` looks like it came from nowhere |
| `ease-in` on dropdown | `ease-out` + custom curve | `ease-in` delays the moment the user watches most; feels sluggish |
| `transform-origin: center` on popover | `var(--radix-popover-content-transform-origin)` | Popovers scale from their trigger, not center (modals are exempt) |

#### Part 2 — Verdict (REQUIRED)

Group remaining commentary by impact tier, highest first. Omit empty tiers.

1. **Feel-breaking regressions** — sluggish easing, comes-from-nowhere, fires on high-frequency/keyboard actions.
2. **Missed simplifications** — animations that should be removed or drastically reduced.
3. **Performance** — non-GPU properties, dropped-frame risks, recalc storms.
4. **Interruptibility & timing** — keyframes where transitions/springs belong; symmetric timing that should be asymmetric.
5. **Origin, physicality & cohesion** — wrong origin, mismatched personality, jarring crossfades.
6. **Accessibility** — reduced-motion and pointer/hover gating.

Close with an explicit decision:

- **Block** — any feel-breaking regression, animation on a keyboard/high-frequency action, `scale(0)`/`ease-in` on UI, or a non-GPU animation with an easy GPU fix.
- **Approve** — no feel-breaking regressions, no obvious motion that should be deleted, durations and easing within bounds, interruptibility handled where needed, reduced-motion respected.

Be specific and cite `file:line`. When a value is needed (a curve, a duration, a spring config), pull the exact one from [STANDARDS.md](STANDARDS.md) rather than approximating.

### Guidelines

- Prefer CSS transitions/`@starting-style`/WAAPI for predetermined motion; JS/springs for dynamic, interruptible, gesture-driven motion.
- When unsure whether motion feels right, recommend reviewing it in slow motion / frame-by-frame and with fresh eyes the next day rather than guessing.

---

## design-it

> **Descrição original:** Routes frontend design tasks to 48 specific UI styles. Triggers for websites, app screens, or UI components requesting a specific aesthetic.
>
> **Fonte:** `self`  
> **Autor:** community

This is the main entry point for the **design-it** skill system. Instead of falling back to generic "AI slop" aesthetics, you have access to 48 distinct, deeply opinionated design styles.

### When to Use
Use this skill when a user requests building any frontend interface (website, app screen, UI component) and you want to apply a specific, opinionated design aesthetic instead of generic defaults.

### How to Use This Skill

When a user asks you to build a frontend interface (web or app):
1. **Identify the Style (Fuzzy Matching)**: Look for keywords in their prompt (e.g., "minimal", "glass", "retro"). You do not need an exact match. Use your semantic understanding to map their request to one of the 48 styles. For example:
   - "Apple style" or "VisionOS" -> `spatial-design`, `bento-ui`, or `glassmorphism`
   - "Windows 8" or "Metro" -> `tile-design`
   - "Terminal" or "Hacker" -> `sci-fi-interface` or `brutalist-typography`
   - "Bauhaus" or "Clean" -> `swiss-design`
   - "Cyber" or "Matrix" -> `cyberpunk-ui`
   If they don't specify a style, choose one that best fits the project context.
2. **Read the Style Reference**: Check the **Style Index** below to find the correct path for the chosen style. Use the `view_file` tool to read its specific `SKILL.md`.
3. **Pick a Palette**: If the user explicitly defined a theme or colors, **use their requested colors**. If they did NOT specify colors, you MUST choose from one of the **10 Universal Palettes** below.
4. **Execute**: Write the code following the specific principles of the chosen style. Do not blend styles unless requested.

### Universal Color Palettes
If the user provides their own colors, use them. Otherwise, you MUST use one of these 10 award-winning, highly sophisticated palettes to avoid generic neon/purplish gradients. When picking a palette, stick to these exact hex codes and establish CSS variables for them.

1. **Yacht Club** (Nautical / Classic Elegance)
   - Backgrounds: `#F9F6F0`
   - Primary Text/Accents: `#1B2A49`
   - CTA/Highlights: `#C85A32`
   - Secondary Base: `#E2D8C9`
2. **Desert Mirage** (Warm / Organic)
   - Backgrounds: `#F4EFEA`
   - Primary Text/Accents: `#2D2B2A`
   - CTA/Highlights: `#A65E44`
   - Secondary Base: `#8C8781`
3. **Industrial Chic** (Strong / Minimalist)
   - Backgrounds: `#D1D1D1`
   - Primary Text/Accents: `#111111`
   - CTA/Highlights: `#9A3B3B`
   - Secondary Base: `#757575`
4. **Monochromatic Brown** (Comfort / Nostalgic)
   - Backgrounds: `#D9CBBF`
   - Primary Text/Accents: `#4A362D`
   - CTA/Highlights: `#7A4C3A`
   - Secondary Base: `#948275`
5. **Earth-Grounded Elegance** (Calm / Sustainable)
   - Backgrounds: `#F7F5F0`
   - Primary Text/Accents: `#3A4B3A`
   - CTA/Highlights: `#8A9A86`
   - Secondary Base: `#D3CEC4`
6. **Minimalist Slate** (Professional / Tech)
   - Backgrounds: `#F4F4F9`
   - Primary Text/Accents: `#2B303A`
   - CTA/Highlights: `#5C6B73`
   - Secondary Base: `#C0C5C1`
7. **Midnight Luxury** (Premium / Dark Mode)
   - Backgrounds: `#0A0A0A`
   - Primary Text/Accents: `#F5F5F0`
   - CTA/Highlights: `#B59A5F`
   - Secondary Base: `#1C1C1C`
8. **Sophisticated Neutral** (Upscale / Lifestyle)
   - Backgrounds: `#E6E2DD`
   - Primary Text/Accents: `#1F1C1B`
   - CTA/Highlights: `#524036`
   - Secondary Base: `#B8B0A8`
9. **Warm Tech** (Corporate / Modern)
   - Backgrounds: `#EAEAEA`
   - Primary Text/Accents: `#1C252E`
   - CTA/Highlights: `#C28F79`
   - Secondary Base: `#2C3E50`
10. **Modern Editorial** (Magazine / High Contrast)
    - Backgrounds: `#F9F9F9`
    - Primary Text/Accents: `#121212`
    - CTA/Highlights: `#D44A3A`
    - Secondary Base: `#8F8F8F`

### The 60-30-10 Rule
- **60%**: Backgrounds / Secondary Base
- **30%**: Primary Text / Accents
- **10%**: CTA / Highlights

---

### Style Index (48 Styles)

To use a style, you MUST read its file at `<style-folder>/SKILL.md` relative to this file's directory.

#### Modern UI
- `minimalism`
- `flat-design`
- `flat-design-2`
- `material-design`
- `glassmorphism`
- `neumorphism`
- `skeuomorphism`
- `claymorphism`
- `aurora-ui`
- `bento-ui`

#### Depth & 3D
- `3d-ui`
- `isometric-design`
- `layered-design`
- `floating-ui`
- `spatial-design`

#### Typography
- `swiss-design`
- `editorial-design`
- `typography-first`
- `brutalist-typography`

#### Retro & Historical
- `brutalism`
- `neo-brutalism`
- `retro-design`
- `y2k-design`
- `cyber-y2k`
- `vaporwave`
- `synthwave`
- `frutiger-aero`
- `retro-futurism`

#### Modern Trends
- `dark-mode`
- `monochromatic-ui`
- `gradient-design`
- `duotone-design`
- `color-blocking`
- `soft-pastel`
- `high-contrast`
- `vibrant-maximalism`
- `maximalism`

#### Futuristic
- `cyberpunk-ui`
- `sci-fi-interface`
- `holographic-ui`
- `ai-native-ui`
- `spatial-computing-ui`

#### Data & Product
- `dashboard-design`
- `card-based-design`
- `widget-based-design`
- `tile-design`
- `data-dense-design`
- `command-center-ui`

### Web vs App Implementation
- **Web (React/Vue/HTML)**: Use CSS variables natively. Prioritize CSS grid/flexbox for layout. Use CSS transitions for hover states.
- **App (React Native/Flutter/SwiftUI)**: Map the palettes to the framework's theme engine. Use platform-specific shadows (elevation) and animations instead of CSS transitions. Maintain the core visual principles while adapting to mobile constraints.

### Limitations

- This skill does not replace environment-specific validation, testing, or expert review.
- Stop and ask for clarification if required inputs, permissions, or safety boundaries are missing.

---

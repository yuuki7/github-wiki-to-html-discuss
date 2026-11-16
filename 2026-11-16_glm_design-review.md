# Design review — interim findings (as of 2026-11-16 09:23)

Subject: `/Users/yuuki/github-wiki-to-html/` (commit b335baf). Points already raised (the type-checking round, and the first design-review round) are excluded. Findings confirmed by running code are marked [verified]; static-reading-only findings are marked [static].

## Ruby (lib) design — additions

### R-G [design decision on record] Ignoring subdirectories is intentional and tested
The old version "flattened plus warned on duplicates"; the current one changed to "silently ignore blobs not at the root", and the intent is pinned by the test 'silently ignores pages and assets in subdirectories'. The silence around content loss is deliberate — however, if GitHub Wiki ever comes to support subdirectories, this policy breaks silently too. Keeping the policy behind a one-line warning option would be safer long-term (no immediate change needed).

### R-H [boundary chain] slug vs language-directory namespace collision [static]
Output is `<language>/<slug>.md`. When a slug coincides with a language code (e.g. a page "ja.md"), the site's getStaticPaths emits the original at `path: 'ja'` (root), while the home for language ja is `/ja` ([...path].astro adds a `params: { path: defaultLanguage }` redirect) — **the same params are generated twice** and Astro conflicts. `occupied_slugs` does not include language codes. The trigger is narrow (a page name exactly equal, lowercase, to a language code), but either the module layer must treat language codes as reserved and reject them, or the site layer must define a precedence.

### R-I [robustness] bin/convert.rb's silent degradation chain when site.json is missing [static]
`site = File.exist?(SITE_PATH) ? JSON.parse(...) : {}` — without site.json the build silently runs a degraded configuration (site_language=nil, supported_languages=nil), and the failure surfaces far away, in the Astro content sync (zod: special requires language). It should fail fast on a missing or invalid site.json.

## Astro/CSS design — additions

### A-H [static + verifiable] the 'unknown' date sentinel leaks into date parsers
The module emits 'unknown' as a legitimate value for author/dates (build_row; reachable through a hand-edited META). The site side:
- `ArticleMeta.astro`: `new Intl.DateTimeFormat(...).format(new Date('unknown'))` → Invalid Date → **RangeError crashes astro build** (`formatDate`; fires when showModified becomes true, i.e. created=real date with modified='unknown')
- `BaseHead.astro`: `datePublished: 'unknown'` (invalid as schema.org)
- `sitemap.xml.ts`: `<lastmod>unknown</lastmod>` (violates W3C Datetime)
- `updated(): modified_date ?? creation_date` — zod makes modified_date required, so the `??` is dead (the type is always a string)
Either type the date as a union of ISO and 'unknown' instead of `z.string()` and branch in the site layer, or drop 'unknown' from the module and emit null in the YAML.

### A-I [minor] og:locale carries a bare language code
Per the OGP spec, og:locale is `ja_JP`-shaped. `"ja"` passes with most consumers but is non-conformant.

### A-J [consistency] `<meta name="color-scheme" content="light dark">` vs hardcoded white
The dark declaration stands while page.css hardcodes `#fff` with zero dark styles. Strengthens the C-G finding.

### A-K [minor] `siteUrl = site.url ?? ""` — if url is missing, canonical/og/sitemap silently degrade into relative URLs and break. Fail fast is the right call.

### A-L [minor] ai-assistant.ts CDN version pin
`libraryUrl` hardcodes @0.2.88; the types come from the package.json devDependency. "The two are the same version" is only guaranteed by a comment — updating one side alone makes the types diverge from runtime. A build-time match check, or a single source of truth.

### A-M [minor] ArticleMeta's `utcDay` does not +1 `getUTCMonth()`
It is comparison-only, so the result is correct (the same bias applies to both sides), but as a date string it is wrong. The intent of the arithmetic is hard to read.

### A-N [structure] the `interface Window` augmentation in mathjax-config.ts
The file is not a module (no import/export), so this works as a global augmentation — but adding a single import line silently breaks it. A `declare global` form would be more robust.

### A-O [minor] the `!dist/` in .prettierignore
It re-includes with `!` although nothing excludes it (defensive? intent unclear). Also `npm run format` formats all of dist's HTML on every build — only two files are tracked, so narrowing the scope or dropping dist formatting would both be fine.

## Test coverage (HEAD, 92 tests)

Strong: the translation pipeline (language refusal / NFD / same-language refusal / disappearance), link-index canonicalization, no gap-marker leakage, `<details>` spanning blocks, TOC literals, rename warning, subdirectory ignoring.

Thin: `escape_url_path` (special-character slugs), `split_stream`'s marker-loss fallback, sweeping `.tmp.<pid>` leftovers, routes assigned to aliases, `translation_title` quote handling, `active_languages` ordering.

## Pipeline / repository

- Only two dist files are tracked, `dist/index.html` and `dist/Wikinder.html` (pinned samples). Astro output is deterministic (no timestamps), so there is no diff churn ✓ (YARD's date stamp is stripped by the XXX hack).
- `astro.config.ts`: compressHTML/minify/cssMinify all disabled — consistent with the project value of "readable output" ✓.
- There is no Ruby version pin (no .ruby-version, no `ruby` directive in the Gemfile) — the code assumes Ruby 4.0 (Set in core, Data, etc.), so a pin is recommended for reproducibility.
- translations/README.md documents the data contract clearly — excellent. "files are added and edited by hand (or by a translating agent)" — a description that assumes an agent workflow.

## Additional verification (09:30)

- [verified] `new Intl.DateTimeFormat("en",{timeZone:"UTC"}).format(new Date("unknown"))` → **RangeError: Invalid time value**. The A-H crash path is real. No 'unknown' dates exist in the current md/, though — a latent risk (reachable through a hand-edited META).
- [verified] `<p>aaa</p>` does not leak into any current dist HTML (every page supplies a right-sidebar; Redirect does not go through Page). It remains a template-hygiene problem.
- [verified] Astro output determinism: the change history of the pinned dist/index.html moves only at feature commits — no diff churn ✓.
- dist/en/<slug>.html files are language-redirect stubs to the original (the design of [...path].astro).

### R-P [design] Wiki's default options embed the site's policy
The default value of `ungenerated_pages` is `Set['Home','README','LICENSE','META','_Footer']` ("the same set the conversion CLI passes"). The library default bakes in **this site's policy** of not emitting Home as an article. Fed another GitHub Wiki, Home silently vanishing from the articles would be surprising. A more neutral default is `Set['META']` (only the file the tool itself manages), with the site's policy passed explicitly by bin.

## Additional verification (09:26) — logic inconsistencies in language handling

### R-Q [verified] Language.normalize collapses region variants for only four languages
Measured: en-US→en, de-AT→de, ja-JP→ja, but **ko-KR→"ko-KR"**, pt-BR→"pt-BR". Because of the hardcoded `%w[en ja de ar]`, even after adding a new language 'ko' to site.json, translations/ko-KR/ is refused at build time as "a language the site does not support". Worse, for unknown bases it returns `code` **preserving the original letter case** (directory-name case consistency breaks too). The collapsing knowledge should take site.json's `languages` as its single source of truth.

### R-R [verified] the link index does not normalize NFD/NFC
Measured: against a "café" page registered in NFC, `index_lookup` with an NFD "café" → **MISS** (fold_key only maps spaces to hyphens plus downcase; no NFC normalization). The translations loader normalizes file names to NFC (with a comment mentioning macOS), but wiki page slugs (collect_pages) are not normalized — if macOS git keeps NFD file names, the NFC links authors write pass through without canonicalization. Adding `unicode_normalize(:nfc)` in two places — collect_pages and fold_key — restores symmetry.

### R-S [minor] [static] no 404 page
pages/ has no 404.astro (relying on the static host's default page). For a public site (wikinder.org), a dedicated 404 (language switcher plus a link home) would be kind.

### R-T [minor] [static] CDN dependencies
MathJax (jsdelivr, with SRI) and WebLLM (jsdelivr, version-pinned) carry the body rendering (math!) and the AI feature. Offline or on CDN failure, math pages break. Recording a self-hosting option is recommended.

### R-V [verified, architectural] the converter bakes the site's URL scheme into the output
Measured: the body links of md/ja/Home.md are `](/ja/Wikinder)` — language-prefixed, resolved for this site's routes, **baked into the emitted Markdown itself** (routes_by_language injected into Tags.render). The file layout, `<language>/<slug>.md`, is part of the same scheme. This is coherent as a "self-contained interchange format": any consumer that mirrors the layout gets working links. On the other hand, against the interoperability claim in README/AGENTS — "many static site generators can process" — **the link-resolution policy is Wikinder-specific** (an SSG with a different scheme must rewrite the body links). The alternative design (the converter emits only canonical /slug links; the site resolves languages with a rehype step) is more portable but moves the complexity to the consumer. The problem is not which choice is right — the choice is documented nowhere. It should be stated in README/AGENTS as the interchange-format contract (the language-directory layout, the link-resolution rules, the meaning of translation/original).

### R-W [test coverage] the TS side has zero tests
Ruby has 92 tests. The TS side holds real logic as pure functions (missingVersions, the sitemap's grouping/sorting, pageHref/homeHref, directionFor) yet has not a single test (no vitest). They are already written as pure functions, so the adoption cost is low.

### R-X [minor] the missing Astro 404 page and the CDN dependencies (MathJax is essential for math rendering) — see R-S/R-T.

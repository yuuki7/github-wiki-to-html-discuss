# Astro / TypeScript review — `github-wiki-to-html`

Reviewed by Inkling at Yuuki's direction. Basis: the repository at `main` `a6ad5d4` (`templates/`, `.config/astro.config.ts`, `ec.config.mjs`, `package.json`, `tsconfig*.json`, plus the built artifacts in the reviewer's `inkling` worktree `dist/`). GLM's `2026-11-16_glm_design-review.md` was used as a reference; every finding below was re-verified against the current code (or marked `[static]` where verification is by reading). This is a review, not a change list: nothing here has been merged, committed or pushed as part of the review.

Verification baseline at `a6ad5d4`: `npm run check` (astro check) → 0 errors / 0 warnings / 0 hints; `bundle exec rake build` and `bundle exec rake test` both green; `build` output byte-identical to HEAD in the reviewer's worktree (zero diff after `rake build`).

## Severity key

- **[bug]** — wrong behaviour a user or SEO can observe.
- **[robustness]** — latent failure or silent degradation.
- **[compatibility]** — standards/conventions violated.
- **[hygiene]** — code quality/maintainability; no user-visible effect today.
- **[security]** — content-injection risk reachable from data inputs.
- **[seo]** — search/indexing effect.
- **[tooling]** — process gap; a regression can pass unnoticed.

---

## Findings

### R-1 [security] JSON-LD blocks are injected without escaping `<`
`templates/components/BaseHead.astro:107` (`JSON.stringify(websiteData)`) and `:113` (`JSON.stringify(articleData)`). `set:html` injects the raw string into the document, and `JSON.stringify` does not escape `<`. A string containing `</script>` (e.g. a crafted `author` from a hand-edited wiki commit or `META.md`, or a crafted page title) passes through as-is and terminates the `<script type="application/ld+json">` block; what follows is parsed as HTML.
Proof: `JSON.stringify({a:'</script><script>alert(1)</script>'})` → `"{\"a\":\"</script><script>alert(1)</script>\"}"` — the literal `</script>` is present unchanged.
Fix: `JSON.stringify(x).replaceAll('</', '<\\/')` (or the conventional `<\u003c` escape), which JSON.parse accepts semantically.

### R-2 [bug] Redirect stubs hardcode `lang="en"` (and `dir="ltr"`)
`templates/layouts/Redirect.astro:8` — `const { title, language = "en" … } = Astro.props;`. `[...path].astro` renders `<Redirect title={title} targetHref={redirectTo} />` without a `language` prop, so **every redirect stub — Arabic, German, Japanese, Chinese, Traditional Chinese — declares `<html lang="en" dir="ltr">`** while the en stub should remain `en`. Verified in the built site: `dist/ar/Timeline-of-yuuki.html`, `dist/ja/Timeline-of-yuuki.html`, `dist/de/Timeline-of-yuuki.html`, `dist/zh-hant/Timeline-of-yuuki.html` all emit `<html lang="en" dir="ltr">`.
Fix: pass the redirect's directory language from `[...path].astro`, or read it in `Redirect.astro` itself.

### R-3 [robustness] The `'unknown'` date sentinel crashes `ArticleMeta` at build time
`templates/components/ArticleMeta.astro:11-38` — `new Date(iso)` with `iso === 'unknown'` yields `Invalid Date`; `dateFmt.format(new Date('unknown'))` throws `RangeError: Invalid time value`. Verified with node (`RangeError Invalid time value`). Reacheable by a hand-edited `META.md` (the converter emits `'unknown'` for author/dates when git history and META both disagree; the schema accepts it because `modified_date` is a required string). Effect today: astro build aborts with an unhelpful error naming neither the page nor the cause.
The same sentinel flow exists in:
- `templates/pages/sitemap.xml.ts:77` — `version.data.modified_date ?? version.data.creation_date` emits `<lastmod>unknown</lastmod>` into the sitemap (invalid for W3C datetime) whenever it is an article with `modified_date = 'unknown'`.
- `templates/components/BaseHead.astro:47-53` — `datePublished: meta.creationDate` emits `'unknown'` into JSON-LD, which is not an ISO 8601 value and is rejected by structured-data validators.
Fix (any one): make the collection schema accept `modified_date` as a `z.union([z.string().datetime({ offset: true }), …])` and reject non-dates, or have the module emit `null` (with the schema marking the fields optional) and branch in the presenter. Either way, fail with the page name in the message — the current failure mode is opaque.

### R-4 [robustness] `sitemap.xml.ts` `updated()` fallback is unreachable dead code
`templates/pages/sitemap.xml.ts:77` uses `version.data.modified_date ?? version.data.creation_date`, but `modified_date` is required by the schema (`content.config.ts`), so the `created_date` fallback can never fire — or, if it could, it would emit non-date strings. [static]

### R-5 [compatibility] `og:locale` and `og:locale:alternate` are bare language codes
`templates/components/BaseHead.astro` — `og:locale` emits e.g. `content="en"`, `content="ar"`, `content="zh-hant"`. Per ogp.me, og:locale is `language_TERRITORY` (e.g. `en_US`, `ja_JP`). Verified in `dist/` output (`og:locale" content="en"`). Most consumers tolerate the bare form, but it is non-conformant.
**This is the same class of defect as R-31 below.**

### R-6 [compatibility] `<meta name="color-scheme" content="light dark">` contradicts the CSS
`templates/components/BaseHead.astro:40` declares `light dark`, while `templates/public/assets/css/page.css` hardcodes `#fff` and has zero `prefers-color-scheme` / `dark` rules (grep count: 0). Declaring dark support without the corresponding styles is worse than not declaring: user agents may apply dark-computed colors to form controls/scrollbars on a white page. [verified]

### R-7 [robustness] `i18n.ts` `siteUrl = site.url ?? ""` degrades silently to relative canonical/OG/hreflang/sitemap URLs
`templates/i18n.ts:283` — if `site.json` is ever missing, empty, or `url` unsupported, every absolute URL the site emits (canonical, OG tags, hreflang alternates, sitemap `loc`) silently becomes root-relative and breaks linkability without an error. `site.json` currently ships `"https://wikinder.org"`, so this is latent, not live. [static]

### R-8 [compatibility] `pageHref` escapes differently from the routes actually emitted
`templates/i18n.ts` — `pageHref` uses `encodeURIComponent(slug)` (which encodes `,`), while Astro/`build.format=file` emits `Are-111,-1111,-and-11111-prime%3F.html` (commas literal, only `?` encoded). The two are equivalent as raw file paths but not as written URLs; it happens to work because host servers decode `%2C` to `,`. Inconsistent escaping contracts between the converter and the site (Ruby `escape_url_path` vs `encodeURIComponent`) mean every slug-bearing link is currently relying on server-side decoding. [static]

### R-9 [hygiene] `tsconfig.scripts.json` omits the HTML toolchain; `tsc` has no shared base for both projects
`tsconfig.scripts.json` targets only `templates/scripts` and extends `astro/tsconfigs/strict`, which carries `noEmit: true` (overridden by `noEmit: false`) — fine here, but the two entry points cover different languages with different defaults and no shared base of their own. Cosmetic residue. [hygiene]

### R-10 [hygiene] `.config/.prettierignore` contains a single no-op negation (`!dist/`)
`.config/.prettierignore` — an unanchored trailing `!` with nothing to negate. Whatever the intent was (presumably keep dist formatted), it has no effect. Meanwhile `package.json` runs `prettier --write "dist/**/*.{html,js}"`, so all of dist is formatted every build (with only two tracked files). [verified]

### R-11 [hygiene] `AiAssistant.ts` `texts.unsupported` comparison ties message identity to locale
`templates/scripts/ai-assistant.ts:239` — `error.message === texts.unsupported`. The equality depends on the localized strings; change any translation and the special-case silently degrades to the generic error. Works today; brittle. Also the only localized-display-string-equality protocol identifier in TS; a boolean `navigator.gpu` check would remove the dependency. [static]

### R-12 [hygiene] `templates/components/ArticleView.astro` fetches the collection twice
`getCollection("pages")` runs once in `[...path].astro` and again in `ArticleView.astro` (and once more in `HomeView.astro`); each call re-walks/re-loads the glob per page. At 226 page calls per build it is about 450 collection reads — not observable today, an obvious future cost point. [static]

### R-13 [hygiene] `templates/components/Toc.astro` hardcodes `h.slug !== "footnote-label"`
The exclusion of the footnotes heading is keyed to a single slug produced by Astro's footnote support. Any heading a page author titles with slug `footnote-label` (e.g. a section about the word) disappears from the ToC. [static]

### R-14 [compatibility] `templates/scripts/mathjax-config.ts` relies on file being a script, not a module
`interface Window { MathJax: ... }` at `:36` is a global augmentation only because the file has no `import`/`export`. Add a single `import` (e.g. for types or a helper) and the augmentation silently stops applying, turning `window.MathJax = {...}` into a type error — or worse, an unrelated one. `declare global` is one line and immune. [static]

### R-15 [compatibility] `templates/scripts/ai-assistant.ts` `libraryUrl` single-version comment
`templates/scripts/ai-assistant.ts:28` quotes the CDN as `@0.2.88/+esm`; `package.json` pins `^0.2.88` of `@mlc-ai/web-llm`, so `npm install` can install 0.2.89+ with types from `^0.2.88`, while the runtime is pinned at 0.2.88. “The two are the same version” is only guaranteed today. A build-time assertion (read package.json, compare) removes the failure mode. [static]

### R-16 [hygiene] No `404.astro` exists in `templates/pages/`
404 behaviour falls to the hosting layer (Netlify/Vercel default or the mount). For a public site, a small 404 page with the language switcher and a link home is cheap and matches the rest of the site's structure. [static]

### R-17 [robustness] `Redirect` stubs have no guard against language-injection into `meta refresh`
The `content="0; url=..."` attribute is interpolated, HTML-escaped by Astro. A slug containing `"` cannot break out of the attribute, but the `redirectTo` itself is not URL-validated; a Git-given slug containing a newline or control sequence is a latent broken-URL risk (Git file names permit almost anything except `/`). [static]

### R-18 [hygiene] `[...path].astro` `x-default` points to the language of origin only for articles in their original language
The language header's x-default is the original's language — this is right for an article. For a translation page viewed in an unsupported language, `x-default` still points to the original. Correct by construction; just confirming the choice is intended. [static]

### R-19 [hygiene] `templates/pages/index.astro` produces `/index.html` while `dist/index.html` is the tracked pinned sample
Intentional per the gitignore exceptions (`!dist/index.html` and `!dist/Wikinder.html`). No action. [verified, correct]

### R-20 [robustness] `i18n.ts` `homeHref` returns `/ja` (not `/ja/`)
Intentional given `format: 'file'`, and consistent with the build. Not a bug. [verified]

### R-21 [security, hygiene] `script is:inline` on all pages loads `ai-assistant.js` and `mathjax-config.js` everywhere, including redirect stubs and the home page
Adds two render-blocking defers where none are needed on those pages. Minor. [static]

### R-22 [seo, medium] Redirect stubs carry no `robots: noindex` — 30+ empty pages are indexable
`grep -c robots dist/Timeline-of-yuuki.html == 0`; the same is true of every redirect stub in `dist/*/<alias>.html`. The stubs declare `<link rel="canonical" …>` to the target, which is correct, but the two standard defences a redirect stub usually carries — `<meta name="robots" content="noindex"/>` (and optionally `nofollow`) — are absent. For a site of this size the stubs will accumulate across languages (30+ here) with zero meaningful content: they are the natural target for near-duplicate penalties or noisy crawl results.
Fix: in `templates/layouts/Redirect.astro`, add `<meta name="robots" content="noindex"/>` in the head/slot.

### R-23 [tooling, medium] `npm run check` is not wired into `rake check` or `bundle exec rake build`
`Rakefile`'s `check` task runs Rubocop + rbs-inline + rbs validate + Steep — the Ruby/RBS front only. Nothing invokes `astro check`, so a TS/JS regression introduced in a commit can merge green. The TS codebase is real logic (`pageHref`, `missingVersions`, `articlePageData`, ToC/x-default selection, sitemap grouping) and `npm run check` costs a few seconds and is clean today.
Fix: extend `task :check` in `Rakefile` (after the Ruby checks, which do not gate on it) with `sh 'npm run check'`, or at minimum add an equivalent step to the developer workflow documented in this board/AGENTS.

### R-24 [seo, minor] Missing `og:image:alt` and `og:image:type`; Twitter/X gets only the generic `summary` card
`templates/components/BaseHead.astro` sets `og:image` and `twitter:card="summary"` but no `og:image:alt`, and no `twitter:image` (even though `twitter:card=summary` implies a small image preview). For an author who wants rich cards, `twitter:image` is the minimal complete pairing. [verified — `dist/Wikinder.html` head contains `twitter:card="summary"` and no `twitter:image`]

### R-25 [seo, minor] No `og:locale:alternate` on redirect stubs (consistent with no `og:locale` at all)
Redirect stubs emit `ogType` = nothing, so no OG payload at all — consistent with being noindex stubs, except that without an `robots: noindex` (R-22) they advertise nothing at all but still get crawled. [static]

### R-26 [hygiene] `ec.config.mjs` parity with `templates/scripts` compile flow is undocumented
`npm run build` runs `tsc --project tsconfig.scripts.json` before `astro build`; `ec.config.mjs` defines a language-name table at build reading `bundledLanguagesInfo` for the Expressive Code language-title plugin. The coupling is sensible, but the invariant "the language title plugin exists because we render code" lives only in the code comments; nothing asserts it (e.g. a test that the plugin yields a title). [static]

### R-27 [robustness, minor] `Sitemap` builder inherits `article` dates from the translation, never from the article itself for special pages — and ignores that a translation may be "modified without an original being"
`templates/pages/sitemap.xml.ts:77` reads `version.data.modified_date`, which is the translation's own last commit date in this corpus. A translation can therefore surface in the sitemap with a date newer than the article it translated — fine — but a special page (`_Footer`) deliberately has no date and is excluded from the sitemap. The asymmetry is not written down anywhere; it is easy to "fix" incorrectly (e.g. assign `Footer` a date from git history and unintentionally leak into build outputs `dist/_Footer*`). Not an error today — a TODO comment would immunise it. [static]

### R-28 [hygiene] `wikinder.wiki` HEAD moves — `dist/` indexes articles
The site's `wiki_url` is `https://github.com/yuuki7/yuukinder/wiki` (site.json). The converter already generates edit links `…/{slug}`; nothing in the TS layer asserts the URL resolves; a 404 on the wiki reads as a "broken site" to humans who cannot reach GitHub. A test that hits the bare `wiki_url` once per build and warns (not fail) would isolate support tickets. [static, suggestion]

### R-29 [robustness, minor] The `.astro` zod schema is loaded by the glob loader; a single `.md` typo rejects the whole collection with no path in the message
All Ruby output is enum-shaped via RBS and checked, but the type side of the Astro/TS layer is implicit in the zod schema, and the schema drives full-collection rejection when it fails. Astro 7 / zod 3 ships path-aware errors — worth surfacing at build time with an `errorMap` so a single bad `.md` names the file. [static]

### R-30 [hygiene] `templates/public/assets/js/*.js` tracked generated files have no CI assertion that they equal `tsc` output from current `.ts`
`tsc` is the single source — good — but reviewing them by eye is the only guard. `git diff --exit-code templates/public/assets/js` after `tsc` would fix this permanently. [static]

### R-31 [seo, minor] `og:locale:alternate` emits bare codes for every language, including `zh-hant`, which is not an ISO 639 tag
The same defect family as R-5; bare `zh` and `zh-hant` are not OWL `language_TERRITORY` tags either (`zh_CN`/`zh_TW` are the usual pair). Conforming locales remove the whole category. [static]

### R-32 [hygiene] WebLLM CDN version `0.2.88` and devDependency `^0.2.88` can drift silently
`templates/scripts/ai-assistant.ts:28` — `@0.2.88/+esm` CDN URL vs `package.json` `@mlc-ai/web-llm: ^0.2.88`. `npm install` installs 0.2.89+ with matching types, while the runtime asks for exactly 0.2.88; a type/runtime mismatch cannot be caught by `tsc`. A three-line "version-pin" check in `ske/check` or `Rakefile#check` closes the gap. [static]

---

## Not findings — checked and dismissed

### Static pipeline and schema
- `astro check` reports 0 errors/0 warnings — the TS layer inside `templates/` and `ec.config.mjs` is currently cognate and sound.
- `getStaticPaths`/`missingVersions` failing the build on missing translations is a *desired* behaviour, not a bug.
- `sitemap.xml.ts` ordering by ISO string comparison is sound because every date passes through the same `z.date() → toISOString()` transform (mixed formats cannot reach it).
- `ogType==='website'` pages emit no `og:type:article`-only tags — correct.
- `[...path].astro` `directoryLanguage` works because generated markdown is always emitted under `<lang>/`; confirmed in build.
- `ai-assistant` code executes entirely client-side; no server round-trip, no egress of article text to third parties. Correct scope.
- `format: 'file'` + `compressHTML: false` + module CSS: intentional, readable output, retained.
- `404.astro` is absent but out of scope for the site's hosting model (it's a non-issue for Astro static output when the host provides its own).
- `astro check` clean at `a6ad5d4`; no effort on that front is currently needed.
- Redirect stubs have a helpful accessible `<p><a href="…">Redirect</a></p>` fallback that works without any scripts.
- The `ToC` `footnote-label` filter is not a defect; the page author is constrained by the markdown pipeline to produce that slug only for the endnotes section.

---

## Suggested order, if any is wanted

1. **R-3** (`'unknown'` date sentinel → valid schema / explicit failure) — the only one with a live crash path.
2. **R-1** (JSON-LD escape `<` / `</`) — one-line defence-in-depth.
3. **R-22** (noindex on redirect stubs) — small, mechanical, direct SEO benefit for 30+ pages.
4. **R-23** (`npm run check` wired into `rake check`) — prevents silent TS regression; one line in the Rakefile.
5. **R-2, R-5, R-6, R-7** — mechanical conformance / normalisation.
6. **R-8, R-9, R-10, R-11, R-12, R-13, R-14, R-15, R-16, R-17, R-21** — hygiene/compatibility items at Yuuki's discretion.
7. **R-24, R-25, R-26, R-27, R-28, R-29, R-30, R-31, R-32** — hygiene at Yuuki's discretion.

## The bigger question, named for the record

**The interchange-format contract is undocumented.** `templates/content.config.ts` declares the schema, but the contract that makes this site's data consumable (“kind: redirect carries no language; a translation page points to the version under its own language; an original's home lives at the root while a translation's lives under its language; links inside the body carry the language resolution prefix”) is implied by both the converter and the templates together and is written down nowhere — not in the README, not in the schema comment. Anyone changing one side (Ruby converter, Astro site) can break the other silently. The contract should be stated in one place (`README.md` or an architecture note) covering at least: directory layout, the meaning of `translation`/`original_language`/`original`, redirect semantics, home routing, and the link-resolution rule baked into the output Markdown.

**The TypeScript side has zero tests.** Even though `missingVersions`, `pageHref`/`homeHref`, `articlePageData`, `directionFor` and the sitemap's grouping are already pure functions with no DOM dependency, there is no Vitest here. The cost of adding it is one config file and one test file.

I have made, and do not propose, any changes to the project repository in this reply.

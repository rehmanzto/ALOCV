# ALOCV

Bangladesh-first, free, bilingual (Bangla/English) professional CV builder.
See the product spec for the full picture — this README covers the codebase.

## Quick start

```bash
npm install
cp .env.example .env.local   # optional — the app runs without this too
npm run dev
```

Open http://localhost:3000. The landing page, onboarding wizard, and
profile placeholder all work immediately with **no configuration** —
Supabase calls are skipped gracefully and job sector/type data comes from
a static fallback until you connect a real project.

## Connecting Supabase (when you're ready)

1. Create a project at supabase.com.
2. Run the SQL in `supabase/migrations/` against it, in order (SQL editor,
   or `supabase db push` if you're using the CLI).
3. Copy your Project URL and anon key into `.env.local`.
4. Restart the dev server. Onboarding will now read live `job_sectors` /
   `job_types` rows instead of the static fallback — no code changes needed.
   `/login` and `/signup` will also switch from the "accounts aren't set
   up yet" placeholder to real Supabase Auth forms automatically (see
   `src/lib/supabase/is-configured.ts` — the same flag gates both).
5. In your Supabase project's Auth settings, decide whether email
   confirmation is required. If it is, signup shows a "check your email"
   screen instead of signing the person in immediately — the code already
   handles both cases (`src/app/signup/page.tsx`).

Auth is wired up (signup, login, logout, the Header reflecting session
state via `src/lib/supabase/use-session.ts`) but the CV draft itself
still lives in `localStorage`, not tied to the signed-in user yet —
that's the next piece, not this one.

## Connecting the AI Assistant (when you're ready)

Set `ANTHROPIC_API_KEY` in `.env.local`. Until then, the mock provider
handles all "improve wording" requests with basic formatting cleanup, so
the full Accept/Edit/Dismiss UI works without any external service.

## Architecture

- **Next.js 16 App Router, TypeScript, Tailwind v4.**
- **Profile-first**: `career_profiles` is the source of truth; a `cvs` row
  is a template + section-visibility choice referencing it, never a copy
  of the data. See `supabase/migrations/0001_init.sql` (`0003_builder_fields.sql`
  adds the columns the builder's form needed — additive only, nothing removed).
- **One CV data model** (`src/types/cv.ts`): the builder form, the live
  preview, both templates, and PDF export all read/write this same shape.
  Currently persisted to the browser (`src/lib/builder/cv-draft-store.ts`,
  `localStorage`) rather than Supabase — the model's field names track the
  DB schema closely so that swap is a mapping exercise, not a rewrite.
- **PDF export uses the browser's native print pipeline**
  (`window.print()`, see `src/lib/builder/download-pdf.ts`), not a
  PDF-generation library. This was a deliberate change after testing:
  `@react-pdf/renderer` was tried first, but its text-layout engine
  scrambled Bangla's complex-script vowel-sign ordering (confirmed by
  extracting text from a generated PDF — verified again against a real
  Chromium print run, which renders Bangla correctly, since it reuses the
  same engine as normal page rendering). The builder hides everything
  except the CV preview at print time via Tailwind's `print:` variant.
- **Data-driven job sectors/types**: read from Supabase when configured,
  from `src/config/job-taxonomy.ts` otherwise (`src/lib/data/job-taxonomy.ts`
  is the single place that decides which). Add a sector with an `INSERT`,
  not a code change. The picker UI (`src/components/shared/sector-picker.tsx`)
  is shared between onboarding and the builder's Job Sector section —
  there's one taxonomy implementation and one picker for it.
- **AI provider abstraction** (`src/lib/ai/`): `AiProvider` interface,
  swappable implementation (`MockAiProvider` / `AnthropicAiProvider`),
  picked by `ai-service.ts` based on whether `ANTHROPIC_API_KEY` is set.
  Called from the client only via `src/app/api/ai/improve/route.ts` — a
  server Route Handler, so the API key is never sent to the browser.
- **Row Level Security everywhere**: every user-owned table has a policy
  scoping it to `auth.uid()`, so cross-user data leakage isn't reachable
  even if application code has a bug.
- **i18n**: `src/i18n/LanguageProvider.tsx` + `src/locales/{en,bn}/common.json`
  for app chrome; `src/components/cv-templates/labels.ts` for the CV
  document's own section headings (kept separate since one is UI copy and
  the other is printed document content). All UI copy goes through
  `t("some.key")` — nothing hardcoded in components.
- **Fonts**: self-hosted Noto Sans + Noto Sans Bengali (`src/fonts/`,
  wired in `src/app/layout.tsx`) — same type family across both scripts
  for consistent metrics, and no runtime dependency on Google's CDN.
- **`useSyncExternalStore` for every localStorage read** (language,
  onboarding draft, CV draft): a plain `useEffect` + `setState` on mount
  either trips React's `set-state-in-effect` lint rule or risks a
  hydration mismatch. The snapshot getters are cached by a version
  counter bumped on write (`getCachedCvDraftSnapshot` in
  `cv-draft-store.ts`) — returning a fresh parsed object on every call
  breaks `useSyncExternalStore`'s reference-equality check and can cause
  excessive re-renders under frequent input (found via automated
  end-to-end testing, not by inspection).

## What's built

- Foundation — project setup, DB schema + RLS, auth plumbing, AI
  abstraction, i18n, design system
- Landing page, `/templates` gallery
- Onboarding (person type -> job sector -> job type -> experience level
  -> review)
- **CV Builder** (`/builder`) — Personal Info (incl. photo), Education,
  Experience, Skills, Job Sector, Template, all backed by one shared
  draft with live preview and autosave to `localStorage`
- Two templates (ATS Professional, Modern) — genuinely different layouts,
  not a recolor, both reading the same `CvDraft`
- PDF export via the browser's print pipeline, verified to render Bangla
  correctly
- **AI Assistant** wired into the builder's Summary and Experience
  description fields — "Improve with ALOCV AI" → Accept / Edit / Dismiss,
  never auto-applied. Calls go through `src/app/api/ai/improve/route.ts`
  so `ANTHROPIC_API_KEY` never reaches the browser; falls back to the
  mock provider (and says so) when it isn't set.
- **Auth** — `/login` and `/signup` (Supabase email/password), Header
  shows the signed-in user + Log out. Both pages show a plain "accounts
  aren't set up yet" message instead of a broken form until Supabase is
  configured.
- `/profile` placeholder that reads the onboarding draft (now points into
  the builder as its main CTA)

## What's next (in priority order)

Persist the CV draft to Supabase behind an account (currently
`localStorage` only, even when signed in) → Dashboard (list/manage saved
CVs, needs the above) → Cover Letter / Skill Gap / Job Matcher (Phase 2-3
of the original spec) → broader validation and empty/error-state polish
→ expand the template library past two.

## Commands

```bash
npm run dev      # local dev server
npm run build    # production build
npm run start    # run the production build
npm run lint     # ESLint
npx tsc --noEmit # type-check only
```

## A note on translations

The Bangla strings (UI copy and the seeded job sector/type names) were
machine-drafted for this build and should get a native-speaker review
pass before shipping to real users — wording, register, and regional
term choices are the kind of thing worth a second pair of eyes on.

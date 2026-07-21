# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

**PUMP360 Avaliação** is a physical-assessment (avaliação física) web app for a
personal trainer (Fábio Privati, CREF 128210-G/SP). It lets the trainer capture
a client's full body assessment, compute body composition / metabolism / goals,
save the result to the cloud, and share a read-only mobile report link with the
client.

The entire product is **two standalone static HTML files** — no build step, no
framework, no package manager, no server code. Everything (HTML, CSS, JS) is
inlined in each file. All UI text is **Brazilian Portuguese (pt-BR)**.

| File | Role | Audience |
|------|------|----------|
| `index.html` (~6,500 lines) | The trainer-facing assessment tool: an 8-step wizard that captures data, runs all calculations, renders a dashboard, exports PDF, and saves to Supabase. | Personal trainer (author) |
| `avaliacao.html` (~925 lines) | The client-facing read-only report. Loads a saved assessment by `?token=` from the URL and renders a polished mobile view. | Client (aluno) |

## How to run / test

There is no build and no test suite. To work on it:

- Open the file directly in a browser, **or** serve the folder over HTTP
  (needed for the CDN scripts and Supabase calls to behave predictably):
  `python3 -m http.server 8000` then visit `http://localhost:8000/`.
- To exercise `avaliacao.html` you need a valid `?token=` that exists in the
  Supabase `avaliacoes` table, e.g. `avaliacao.html?token=<uuid>`.
- Verify changes **manually in the browser**. There is no linter, formatter, or
  CI configured. Match the existing code style rather than introducing tooling.

## Architecture & data flow

```
index.html (trainer)                         avaliacao.html (client)
─────────────────────                        ───────────────────────
8-step wizard fills the form
   │
   ├─ live calcs (IMC, dobras, metab, metas)
   ├─ draft autosaved to localStorage
   │
   └─ "Salvar na Nuvem"
        → coletarTodosDados() builds `avaliacaoAtual`
        → supabase.insert into `avaliacoes` { personal_id:'fabio', dados:{...} }
        → DB returns a generated `token`
        → gerarLinkAluno(token) builds the public link
        → link shared via copy / WhatsApp / EmailJS
                                                   │
                                                   ▼
                                    reads ?token= → supabase
                                      .from('avaliacoes')
                                      .select('*').eq('token',token).single()
                                    → render(data.dados)
```

### Backend: Supabase

Both files talk to the same Supabase project directly from the browser using
the JS SDK (`@supabase/supabase-js@2` via CDN):

- URL: `https://kwdlicxmwlhbpppnixec.supabase.co`
- Publishable (anon) key is hardcoded in both files — this is a **public
  client key by design**, not a secret. Row-level security on the `avaliacoes`
  table is what protects the data; do not treat this key as sensitive.
- Single table used: **`avaliacoes`**, with at least these columns:
  `id`, `token` (generated, used as the share key), `personal_id` (currently
  always the string `'fabio'`), and `dados` (JSON blob holding the whole
  assessment).

### The `dados` JSON shape (contract between the two files)

`index.html`'s `coletarTodosDados()` writes this object; `avaliacao.html`'s
`render(d)` reads it. **Keep the two in sync** — if you add/rename a field in
one, update the other.

```
dados = {
  dadosPessoais:  { nome, dataNascimento, idade, sexo, objetivo,
                    nivelAtividade, telefone, dataAvaliacao },
  anamnese:       { patologias, sono, estresse, medicamentos, lesoes },
  antropometria:  { peso, altura, imc, classificacaoImc, rcq, rce,
                    cintura, quadril, circs:{...16 circumferences...} },
  composicao:     { percentualGordura, massaGorda, massaMagra, pesoIdeal,
                    densidade, protocolo },
  metabolismo:    { tmb, get, metaCalorica, ajusteKcal,
                    proteinaG, carboidratoG, gorduraG },
  metas:          { pesoMeta, gorduraMeta, massaGordaMeta, massaMagraMeta,
                    habitos:{h_agua,h_alcool,h_cardio,h_sono,h_proteina,h_carga},
                    mensagem },
  postural:       { ...pill selections per view... },
}
```

Note: `render()` in `avaliacao.html` is defensive and also accepts several
legacy/flat fallback keys (e.g. `d.antrop`, `d.dobras`, `d.imc`) — preserve
those fallbacks when editing so older saved records keep rendering.

### Third-party libraries (all via CDN, no local deps)

- `@supabase/supabase-js@2` — database access (both files)
- `chart.js@4.4.0` — dashboard charts (`index.html` only)
- `jspdf@2.5.1` + `jspdf-autotable@3.5.31` — PDF export (`index.html` only)
- `@emailjs/browser@4` — sending the client link by email
  - EmailJS public key: `FgdEbUE0GfCjfH0Ph`
  - service `service_9bvhclu`, template `template_db66p1p`
- Google Fonts: **Inter**

## `index.html` internals

### The 8-step wizard

Tabs are `<div class="tab-content" id="tab-1">` … `id="tab-8">`, driven by the
`STEPS` array and `goToTab(n)`. Order:

1. **Cadastro** — personal data (name, DOB, sex, goal, activity level, phone)
2. **Anamnese** — health history (pathologies, sleep, stress, meds, injuries)
3. **Antropometria** — weight, height, IMC, RCQ/RCE, 16 circumferences
4. **Dobras** — skinfold measurements → body composition
5. **Postural** — posture assessment via photo overlay + pill checklists
6. **Metabolismo** — TMB/GET, calorie target, macro split
7. **Metas** — 3-month goals + habits + motivational message
8. **Dashboard** — summary, charts, PDF export, "Salvar na Nuvem", share

### Key domain logic (functions to know)

- `calcIMC()`, `imcClass()` — BMI and its classification band
- `calcRCQ()`, `rcqClass()` — waist-to-hip ratio
- `calcBodyComp()` — body composition from skinfolds
- `PROTOCOLS` — six skinfold protocols: `pollock7`, `pollock3`, `durnin`,
  `faulkner`, `guedes`, `petroski`. Some (`pollock3`, `guedes`) use different
  skinfold sites for men vs women (`dobrasM`/`dobrasF`). `updateProtocols()`
  rebuilds the skinfold input fields when the protocol or sex changes.
- `siri()`, `durninCoeff()`, `percGClass()` — body-fat % from density
- `calcMetabolism()`, `renderMetabStrategy()`, `renderMetabMacroCards()` —
  metabolism tab; state persists in `_metabState` across live slider updates
- `VIEWS` + `POSTURAL_CHECKS` + `PC_STATE` — postural module. Four views
  (frontal, posterior, lateral_d, lateral_e); each has segments of pill options.
  Selected pills live in `PC_STATE`; `DIAGNOSTICOS_MAP` maps selections to
  postural diagnoses.
- `HABITS_DEF` / `GOALS_HABITS` — habit toggles on the Metas tab. Habit IDs
  (`h_agua`, `h_alcool`, `h_cardio`, `h_sono`, `h_proteina`, `h_carga`) must
  match the `HABITS_LABELS` map in `avaliacao.html`.
- `coletarTodosDados()` — the single serializer that assembles `avaliacaoAtual`
  before saving. **This is the source of truth for the `dados` schema.**

### Client / persistence layer

- **Alunos (clients):** `getAlunos()`/`saveAlunos()` keep a client roster in
  `localStorage` under `pump360_alunos`; assessment history under
  `pump360_history`. `getOrCreateAluno()` de-dupes by lowercased name.
- **Draft autosave:** `saveDraft()`/`loadDraft()` persist the in-progress form
  to `localStorage['pump360_draft']`; `debounceDraftSave()` throttles writes on
  input. New assessments clear this key.
- **Cloud save:** `salvarNoSupabase()` inserts the row and stores the returned
  `token`/`id` back onto `avaliacaoAtual`.
- **Sharing:** `gerarLinkAluno()`, `exibirModalLink()`, `copiarLink()`,
  `compartilharWhatsApp()`, `enviarLinkEmail()`.

## Conventions

- **Language:** all UI strings, comments, variable names, and toasts are in
  pt-BR. Keep new user-facing text in Portuguese.
- **Style:** vanilla JS, no modules, functions on the global scope, inline
  `onclick=` handlers in the HTML. CSS uses custom properties defined in
  `:root` — the brand palette is dark background + gold accent
  (`--gold:#c9a84c`), status colors `--success`/`--warning`/`--danger`.
  Reuse existing CSS variables instead of hardcoding colors.
- **Single-file rule:** everything stays inline in the one HTML file. Do not
  split into separate JS/CSS assets or add a bundler unless explicitly asked.
- **Defensive DOM access:** existing code guards element lookups
  (`document.getElementById('x')?.value || '—'`) and wraps calc calls in
  try/catch. Follow the same pattern — the wizard renders tabs lazily so
  elements may not exist yet.
- **Numbers:** `v(id)`, `n(id)`, `pf()`, `pi()` are the small parsing helpers;
  `'—'` is the standard "no value" placeholder throughout.

## Gotchas

- **Two files must agree.** Any change to what `index.html` saves in `dados`
  needs a matching change in `avaliacao.html`'s `render()`, and vice versa.
- **Whole-file edits are large.** These files are big and monolithic; use
  targeted edits and search for the specific function rather than rewriting
  sections wholesale.
- **`index.html` has a UTF-8 BOM** at the start of the file — preserve it.
- The Supabase anon key and EmailJS public key are meant to be public; don't
  "fix" them by removing or rotating them without understanding the RLS setup.

## Git workflow

- Default branch: `main`. Historic commits are bulk "Add files via upload"
  from the web UI (no local dev history) — this repo is edited as uploaded
  static files.
- Commit with clear, descriptive messages. Only push to the branch you were
  asked to work on; do not open a PR unless explicitly requested.

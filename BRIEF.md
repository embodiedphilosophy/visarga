# Visarga — Build Brief for Claude Code

*Prepared 26 October 2026 for Jacob Kyle (Embodied Philosophy / Tarka). Hand this whole file to Claude Code at the start of the build.*

---

## 0. How to use this brief (for Jake)

1. On GitHub, create a new empty repository called **`visarga`** under the `embodiedphilosophy` organisation.
2. Open Claude Code in that repository.
3. Drag this file into the repo (or paste it) and save it as **`BRIEF.md`** in the top-level folder.
4. Paste this as your first message to Claude Code:

> Read BRIEF.md in full. It is the spec for this project. Start with Phase 1 only. Before writing code, reply with (a) your proposed tech stack and folder structure, (b) the database schema as a list of tables and fields, and (c) any questions about the "Open decisions" in section 2 that block Phase 1. Wait for my OK before building.

5. When Claude Code asks you to set up accounts (Vercel, database, Stripe), it will tell you which buttons to press. You don't need to apply patches or edit code yourself; ask it to make every change directly in the repo and deploy it.

---

## 1. What Visarga is

**Visarga** is a website for **open, scholar-edited English translations of Sanskrit philosophy**: verse by verse, with the Sanskrit, the notes, and every alternative rendering the editors have weighed.

- **The texts are free to read, always.** Anyone can read the current version of every translation, including the Sanskrit and the footnotes.
- **Translations are revisable.** Approved scholars on the **editorial staff** propose alternative renderings and contribute footnotes. Each text has one **general editor** who alone accepts, revises, or declines proposals. Accepted changes are released together as numbered, citable **versions** (v1.0, v1.1…).
- **Members pay for the tools**: word-by-word analysis, a lexicon, all renderings on record, version comparison, Sanskrit search, private notes, exports.
- **Other revenue**: institutional licences, paid live **reading rooms** (small cohorts reading a text with its general editor), **print editions**, and **patrons** who "adopt a text" to fund an editor's work.

Visarga is the reading and membership front end for the **Tarka Translations** programme. Tarka (tarkajournal.com) remains the editorial home.

**The name.** The visarga is the Sanskrit sign ः (two vertical dots), whose literal meaning is "release, sending forth"; in Abhinavagupta's phonematic theory it is Śiva's creative emission. The two dots, one indigo and one red, are the logo.

**The first text** is Kṣemarāja's ***Pratyabhijñāhṛdaya*** (20 sūtras with commentary), from Jake's existing translation (see section 9).

---

## 2. Decisions

### Made

| Decision | Choice |
|---|---|
| Name | Visarga |
| Who approves changes | Only the text's general editor. Other editors propose and discuss. |
| Business model | Open core: free texts; paid tools, teaching, print and patronage |
| Release model | Accepted changes collect into the next numbered version; each version is frozen and citable |
| First text | Pratyabhijñāhṛdaya (Jake's translation) |
| Look | Riso, two-ink: Tarka indigo + riso red on warm paper; Syne / EB Garamond / Courier Prime (section 7) |
| Hosting | Vercel, GitHub repo `embodiedphilosophy/visarga`, like the EP and Tarka sites |

### Open (ask Jake before building anything that depends on these)

1. **Licence** for the translations (CC BY-SA 4.0 is the default assumption). If an open licence is chosen, nothing in the *text* may be paywalled, only tools and services.
2. **Who can see drafts**: all approved editors, or only editors approved for that text?
3. **Partial texts**: may a single chapter of a long work be released as a numbered version?
4. **Where the editorial review queue lives**: inside Visarga under `/desk`, or in the separate Tarka Desk app (repo `tarka-desk`, desk.tarkajournal.com), which owns people and submissions for Tarka. *Default for Phase 1: build it inside Visarga at `/desk`, with a clean API boundary so it can move later.*
5. **Legal entity and payments**: Tarka nonprofit programme vs. an Embodied Philosophy product. This affects which Stripe account to use. *Default: build against a Stripe test account and keep the account configurable.*
6. **Prices** (all placeholders): Member ~$8/mo or $80/yr, reduced rate ~$4/mo; Institution from ~$1,200/yr; reading rooms per cohort.
7. **Domain** (not yet checked or bought).
8. **Source edition** for the Pratyabhijñāhṛdaya (not named in Jake's doc).

---

## 3. Users and roles

| Role | Can |
|---|---|
| **Reader** (anonymous or free account) | Read every published version, notes, citations; sign up for the verse-of-the-week email |
| **Member** (paid) | Everything above, plus word-by-word, lexicon, renderings on record, version compare, Sanskrit search, private notes (shareable to a reading-room group), exports |
| **Institution user** | Member tools, granted by IP range or institutional sign-in (Phase 3) |
| **Editor** (approved applicant, free membership) | Propose renderings/footnotes on texts they're approved for; comment on proposals; see drafts (per open decision 2) |
| **General editor** (one per text) | Everything an editor can, plus accept / accept-with-edits / keep in apparatus / decline; edit the text directly; cut releases |
| **Admin** (Jake) | Manage texts, collections, people, applications, roles, prices |

A person can hold several roles. General-editor status is **per text**.

---

## 4. Core concepts and data model

Use a relational database (Postgres). Suggested tables. Claude Code should refine these and confirm with Jake.

**Corpus**
- `collections`: the 5 collections (slug, number, title, description, sort). Sub-groups (darśanas etc.) as `collection_sections`.
- `texts`: slug, title (IAST), author(s), date (free text, e.g. "11th c."), collection_id, section_id, **tier** (`launch | in_progress | P1 | P2 | P3`, internal only), **wave** (1–5 or null), general_editor_id, public_status (derived, see below), source_edition, licence, description.
- `text_structure`: hierarchical divisions (e.g. adhikāra › āhnika › verse, or simply sūtra 1–20).

**Content (versioned)**
- `segments`: the citable unit (a verse, a sūtra, or a commentary paragraph). Fields: text_id, stable `segment_key` (e.g. `ph.1`, `ph.1.c1` for commentary para 1), kind (`root | commentary | mangala | colophon`), order, parent_key.
- `segment_revisions`: one row per change to a segment. Fields: `devanagari`, `iast`, `translation` (rich text with inline note anchors), editor_id, created_at, proposal_id (if it came from one).
- `notes`: footnotes anchored to a segment revision and a span (character offsets or a token range). Fields: body (rich text), author_id, credit line, added_in_version.
- `renderings`: alternatives for a span. Status: `in_text | apparatus | under_review | declined`, proposer, version.
- `releases`: version number, date, changelog, the **frozen snapshot** (list of segment_revision ids), citation strings, PDF url. A release is immutable once published.

**Study layer (member tools)**
- `tokens`: per segment revision, the word-by-word analysis: surface form, position, lemma/stem, root, grammatical form (e.g. "f. nom. sg."), sense, compound analysis (vigraha) as optional text.
- `lexicon_entries`: headword, gloss, etymology, related forms; tokens link to entries.
- `member_notes`: private notes on a segment; optional sharing to a group (reading room).

**Editorial workflow**
- `editor_applications`: full form contents (section 5.3), texts requested, sample rendering, sample footnote, referee name/email, referee status, decision, reviewer.
- `proposals`: type (`rendering | footnote`), text_id, segment_key, span, scope (`phrase | pada | verse`), current text, proposed text, rationale, sources (list), keep_in_apparatus_if_declined (bool), status (`draft | submitted | accepted | accepted_with_edits | apparatus | declined`), decision note, reply sent to contributor, target release.
- `comments`: threaded discussion on proposals.

**Community and money**
- `people`: name, email, affiliation, position, ORCID, public credit name, roles.
- `memberships`: Stripe customer/subscription ids, plan, status.
- `institutions`: name, IP ranges, contact, licence period, seats.
- `reading_rooms`: text, leader, schedule, seats, price, member price, status; `enrolments`.
- `sponsorships` ("adopt a text"): text, goal, pledged, patrons, credit wording.
- `print_editions`: release_id, format (edition / riso pamphlet), price, store link.

**Public status mapping** (tiers stay internal):

| Internal tier | Public label |
|---|---|
| launch (with a published release) | Open now |
| in_progress | In translation |
| P1 | Next waves (or "Wave n" if assigned) |
| P2, P3 | Editor wanted |

**URLs and citation**
- `/t/{text}`: current version. `/t/{text}/v/{version}`: a frozen version. `/t/{text}/v/{version}/{segment_key}`: a permalink to one verse.
- The citation for each version is generated: *Author, Title, trans. General Editor et al., Visarga v{n} ({year}), {permalink}* plus BibTeX.

---

## 5. Pages and features

Numbers in brackets refer to the mockups on Jake's design canvas (boards 1–15). The canvas is private, so the descriptions here are the source of truth.

### 5.1 Public site

- **Home** [12]: top banner; nav (Collections, Read, Reading rooms, Editions, Membership, Sign in). Hero headline in EB Garamond: "Read śāstra *with its readers.*" Beside it, the visarga mark: two large overlapping circles (indigo above, red below, multiply blend) on a red halftone panel, with a tilted paper label reading "विसर्ग · visarga · ः · release, sending forth". Then: a strip with three columns (Free / For members / Together); five collection tiles as colour blocks; an **Open now** list with author, editor, version and status tag for each text, followed by a wave-2 strip; a dark "How an edition is made" band with three numbered steps and an "Apply to the editors" button; verse of the week (Devanāgarī, IAST, English, one credited footnote); a reading-room teaser; and a footer colophon: "Set in Syne, EB Garamond & Courier Prime. Two inks."
- **Corpus / collections** [7, 8, 9]: a collections index with counts and a status bar per collection. Each collection page has its sub-groups (e.g. darśana tabs), and a text row or card for each text with author, date, one-line "why it matters" and status tag. Pratyabhijñā page: a "Where to begin" strip. Philosophy page: an "editor wanted" call per shelf and "Read against" cross-links.
- **Release waves** [10]: the five waves with their texts, plus open calls for general editors.
- **Reader** [2, 13]: see 5.2.
- **Revision history** [6]: version timeline (upcoming / current / drafts), changes per version with contributor credits, "Read this version", "Compare", PDF, citation box, contributor list.
- **Apply to the editors** [3]: see 5.3.
- **Membership** [14]: four tiers (Reader free, Member, Institution, Patron) plus "Texts seeking patrons" cards showing funding progress, "Where the money goes", and "Editors read free".
- **Reading rooms & editions** [15]: cohort cards (dates, length, leader, level, price, seats, reserve / waitlist), a members-only rolling "Reading Sanskrit through śāstra" room, print editions and a riso pamphlet series.

### 5.2 The reader (the heart of the site)

For each segment:
1. **Devanāgarī** (`lang="sa"`, Noto Serif Devanagari)
2. **IAST**, with each word a button (`lang="sa-Latn"`)
3. **English translation** with superscript note numbers
4. **Commentary** block (for texts with a commentary), collapsible and paragraph-segmented
5. **Notes**, numbered, each with a credit line: "— Name, role/institution · added vN"

Around it:
- A location bar: breadcrumb (collection › text › division), segment jump buttons, a version selector, general editor and contributor count.
- Display toggles: Devanāgarī / IAST / Notes / Open proposals.
- **Member tools** (marked with a red "Members" tag; free users see a soft upsell, not a hard wall):
  - Word-by-word table (word, form, root/stem, sense) and the compound analysis
  - Lexicon panel for the selected word
  - Renderings on record for the selected span (in text / apparatus / under review)
  - Private notes, optionally shared with a reading room
  - Compare versions (side by side, with changes highlighted)
  - Search across the corpus in Sanskrit (IAST, Devanāgarī, or Harvard-Kyoto input) and English
  - Export a passage: PDF; LaTeX (reledmac), TEI XML and BibTeX in Phase 2
- A citation box: copy, BibTeX, history.
- For editors: "Propose a rendering" and "Add a footnote" buttons on each segment.

### 5.3 Editorial workflow

- **Application form** [3]: name, email, affiliation, position (faculty / postdoc / doctoral / master's / paṇḍita or traditional training / independent), ORCID (optional), publications link, a training description, texts wanted (checkboxes), "would consider being a general editor", a **sample rendering** of a set passage plus one footnote with reasoning, a referee's name and email, and agreement to the licence and to the general editor's final say. After submitting: an automatic email to the referee, then the application goes to the review queue for the general editor of each chosen text.
- **Propose** [4]: select a span in the reader; choose rendering or footnote; choose scope (phrase / pāda / verse); a live **preview in context** (the old wording struck through, the new one inserted); rationale; sources (one per line); a "keep in apparatus if not adopted" checkbox; save draft or send. A side panel shows earlier proposals on the same span and comments.
- **Review queue** [5] (general editor): a list of proposals and applications with filters. The detail view shows the in-context diff, rationale and sources, the discussion, and a decision (accept / accept with my edits / keep in apparatus / decline). An **editable reply template** goes to the contributor by email. Accepted items are queued for the next release.
- **Releases**: the general editor reviews queued changes, writes a changelog and publishes vN. This freezes the snapshot, generates citations, and notifies contributors and followers.
- **Corpus planner** [11] (admin): a table of all texts with tier, wave, general editor and an "English translation exists?" column; counts by tier; filters by collection and "no general editor"; and CSV import of the corpus list.

### 5.4 Email

Use **Kit** (Embodied Philosophy's existing email provider) for: verse of the week, release announcements, reading-room marketing. Transactional email (application received, referee request, decisions, receipts) via a transactional provider (e.g. Resend or Postmark). Claude Code should propose one.

---

## 6. Suggested stack (Claude Code may propose alternatives)

- **Next.js** (App Router, TypeScript) on **Vercel**
- **Postgres** (Neon or Supabase via the Vercel marketplace) with a typed ORM (Drizzle or Prisma)
- **Auth**: magic-link email plus Google sign-in; roles in the database
- **Payments**: **Stripe** Billing (memberships, reduced rate, institutions by invoice, reading-room checkouts, sponsorship pledges)
- **Search**: Postgres full-text search to start. Normalise IAST, Devanāgarī and Harvard-Kyoto to one form so they search alike.
- **Transliteration**: `@indic-transliteration/sanscript` (or equivalent) to generate Devanāgarī from IAST. **Every generated Devanāgarī line must be flagged "unproofed" until an editor checks it.**
- **Rich text** for translations and notes with inline note anchors (Tiptap or ProseMirror)
- **Images and PDFs**: Vercel Blob

---

## 7. Design system

**Feel**: a riso-printed zine meets a scholarly edition. It uses two inks on warm paper, a halftone dot texture, heavy 2px black rules, and small Courier labels in capitals. It should be calm and readable, not shouty: headlines are serif and sentence case, never giant capitals.

**Colour tokens**

| Token | Hex | Use |
|---|---|---|
| `paper` | `#F2EDE1` | page ground |
| `paper-2` | `#E8E1D0` | bands, panels |
| `ink` | `#1B1A17` | text, rules, dark bands |
| `ink-soft` | `#3A3832` | secondary text |
| `indigo` | `#2E3F7F` | ink 1: links, Devanāgarī, member badges, collection blocks |
| `indigo-tint` | `#DCE0EE` | halftone ground (blue) |
| `red` | `#D9432A` | ink 2: fills, buttons, highlights, large type only |
| `red-text` | `#A92F1A` | red for small text (passes contrast) |
| `red-tint` | `#F6D9D0` | halftone ground (red), highlighted words |
| `rule-soft` | `#CFC6B2` | hairlines |

Never set small text in `#D9432A` on paper; use `red-text`. Body text must meet WCAG AA (4.5:1).

**Type** (Google Fonts)
- **EB Garamond**: body text, translations, main headlines (500, sentence case, an italic phrase in indigo for emphasis)
- **Syne**: wordmark (800, capitals), section headings and card titles (600–700, sentence case, 22–34px)
- **Courier Prime**: labels, metadata, tags, buttons (capitals, letter-spacing 1–2px)
- **Noto Serif Devanagari**: all Devanāgarī

**Textures (CSS)**
```css
.ht-blue{background-color:#DCE0EE;background-image:radial-gradient(#2E3F7F 1.1px,transparent 1.4px);background-size:7px 7px}
.ht-red{background-color:#F6D9D0;background-image:radial-gradient(#D9432A 1.2px,transparent 1.5px);background-size:6px 6px}
```
Overlapping shapes use `mix-blend-mode: multiply`, so indigo over red prints purple, as two riso inks would.

**Components**: 2px black-bordered cards; a paper "sticker" panel inset on halftone; status tags (filled indigo = open, outlined = draft, red = editor wanted, black = new); the red "Members" tag; buttons as black or red blocks with Courier capitals; numbered step dots in indigo / red / paper.

**Sanskrit typography**: always mark the language (`lang="sa"` for Devanāgarī, `lang="sa-Latn"` for IAST). Never fake diacritics. Store all text as NFC Unicode.

**Accessibility**: real buttons and links, labelled inputs, visible focus, 44px touch targets, works at 375px width (menus stack, tables scroll in their box), `prefers-reduced-motion` respected.

---

## 8. Build phases

### Phase 1: read the first text (MVP)
- Project setup, database, auth, roles, admin seed (Jake as admin and general editor)
- Collections and texts seeded from the corpus list (section 10). Every text gets a public page with its status, even with no content.
- **Import the Pratyabhijñāhṛdaya** (section 9) as release **v0.1** (editors' draft) and make it readable
- Reader with Devanāgarī / IAST / English / commentary / notes, location bar, permalinks, citation box
- Home, collection pages, text pages, revision history (single version), about / colophon
- Design system in place
- **Done when**: Jake can open `/t/pratyabhijnahrdaya`, read all 20 sūtras with commentary, jump between sūtras, copy a citation, and the site works on a phone.

### Phase 2: editors and releases
- Application form, referee email, application review
- Proposals (rendering and footnote), discussion, review queue, decisions with reply templates
- Releases (v0.1 → v1.0), changelog, version compare, contributor credits
- Corpus planner (admin)
- Exports: PDF, BibTeX; then TEI and LaTeX
- **Done when**: an approved test editor can propose a rendering on sūtra 1, Jake accepts it, publishes v1.0, and the change appears credited with a citable permalink, while v0.1 stays readable.

### Phase 3: members and money
- Stripe memberships (monthly, annual, reduced rate) and gating of member tools with soft upsells
- Word-by-word tokens and lexicon (editor-entered at first; see section 11), private notes, Sanskrit search
- Reading rooms (listing, checkout, enrolment, shared notes group)
- Sponsorships ("adopt a text") with progress
- Institutions (IP-range access, invoice billing, usage report)
- Kit integration (verse of the week, release announcements)

### Phase 4: breadth
- Import the next launch texts (Nyāya-sūtra-bhāṣya āhnika 1, ĪPV Jñānādhikāra 1, ĪPK assembled from Jake's files, Alaṅkārasarvasva)
- Print pipeline (release → PDF for print → IngramSpark/Lulu), riso pamphlet listings
- Recruit general editors by collection

**Not in scope for now**: mobile apps, AI-generated translations, public comments from non-editors, a forum.

---

## 9. First text: the Pratyabhijñāhṛdaya

**Source**: Jake's Google Doc **"Pratyabhijñāhṛdaya - July 18th Translation"**: https://docs.google.com/document/d/1H2L_sYmDuB65bSap6DUJtA-RrmYlZiuvpzNjgucHF-k/edit
(Ask Jake to export it as .docx or Markdown, or to connect Google Drive, if Claude Code can't open it.)

**Structure of the doc**
- Title block, then *oṃ namo maṅgalamūrtaye*
- Two maṅgala verses (English only), then the introductory paragraph
- `## Sūtra N` headings for N = 1–20, each with:
  - a bold IAST sūtra line
  - a bold English translation line
  - `*Commentary:*` followed by one or more paragraphs of English commentary (quotations in italics), ending `‖ N ‖`
- After sūtra 20: two colophon verses and the closing colophon

**Import rules**
- Segment keys: `ph.mangala.1`, `ph.mangala.2`, `ph.intro`, `ph.1` … `ph.20` (root sūtras), `ph.1.c1`, `ph.1.c2` … (commentary paragraphs), `ph.colophon.1`, `ph.colophon.2`, `ph.colophon`.
- Generate the **Devanāgarī** for each sūtra from the IAST and flag it **unproofed**.
- The doc has **no Sanskrit for the commentary**, only English. Leave the commentary's Devanāgarī/IAST fields empty, and show "Sanskrit forthcoming" in the reader.
- Keep Jake's English exactly as written. Convert Markdown emphasis to rich text and strip the `‖ N ‖` markers into segment numbering.
- Record `source_edition` as **"[to be confirmed by Jake]"**.

**Known issues to raise with Jake, not silently fix** (each should become a proposal in the queue once Phase 2 exists):
- Sūtra 8 IAST: *sarvadarśanastitayaḥ* → probably *sarvadarśanasthitayaḥ*
- Sūtra 20 IAST: *…devatācakraśvaratā…* → probably *…devatācakreśvaratā…*
- Sūtra 20 English: the closing "— and this is auspiciousness" isn't in the sūtra
- Sūtra 11 commentary: check the attribution "Śrī Rāma [Kaṇṭha]"

**Seed notes for sūtra 1** (from Kṣemarāja's glosses in Jake's translation; credit "Jacob Kyle, general editor · v0.1"):
1. *citi*: the singular declares she is not bounded by space, time or form, and exposes the unreality of every doctrine of difference.
2. *svatantrā*: distinguishes *citi* from the Vedāntic Brahman and declares her essentially Māheśvara character.
3. *viśvasiddhi*: "accomplishment" covers the universe's production, illumination, and dissolution into the supreme subject.

**Seed word-by-word for sūtra 1** (Jake to verify):

| Word | Form | Root / stem | Sense |
|---|---|---|---|
| citiḥ | f. nom. sg. | citi ← √cit | consciousness as power; Citi |
| svatantrā | adj. f. nom. sg. | sva-tantra | free; self-determining |
| viśva- | stem in compound | viśva | the universe |
| -siddhi- | stem in compound | siddhi ← √sidh | accomplishment, establishment |
| -hetuḥ | m. nom. sg. | hetu | cause |

Compound: *viśvasiddhihetu* = *viśvasya siddheḥ hetuḥ* (genitive tatpuruṣa).

---

## 10. The corpus (for seeding)

About 160 texts so far in five collections, from Jake's "Tarka Translations — Corpus List" (25 Oct 2026). Ask Jake for that document, or a CSV export of it, to seed `texts` with title, author, date, tier, wave and one-line rationale.

| # | Collection | Sub-groups (texts) | Total |
|---|---|---|---|
| 1 | Pratyabhijñā philosophy | single list | 15 |
| 2 | Seminal texts of Indian philosophy | Nyāya 9 · Vaiśeṣika 4 · Mīmāṃsā 6 · Vyākaraṇa 4 · Vedānta 6 · Buddhist 9 · Jaina 5 · Cārvāka & surveys 4 | 47 |
| 3 | Kashmir Śaiva tantras, Trika, Spanda & Krama | Āgamas 9 · Śivasūtra & Spanda 7 · Abhinavagupta 10 · Kṣemarāja & Krama 6 · Śrīvidyā 3 · Śaiva Siddhānta 4 | 39 |
| 4 | Yogasūtra & Sāṃkhyakārikā traditions | Sāṃkhya 12 · Yoga 11 · Yoga in other traditions 11 | 34 |
| 5 | Aesthetics, dharma & Kashmiri setting | Alaṅkāraśāstra 18 · Dharma & authority 1 · Kashmiri setting 4 | 23 |

Tier totals: Launch 4 · In progress 4 · P1 44 · P2 45 · P3 61.
Launch texts: Īśvarapratyabhijñākārikā, Īśvarapratyabhijñāvivṛtivimarśinī, Nyāyasūtra with Nyāyabhāṣya, Alaṅkārasarvasva (general editor wanted). The Pratyabhijñāhṛdaya (P1) is published first as the pilot.

Waves: 1 Launch · 2 the dissertation context · 3 the yoga book · 4 work in progress migrated (Tantrāloka with Viveka, Nyāyakusumāñjali, Śṛṅgāraprakāśa) · 5 breadth via recruited general editors.

---

## 11. Notes and cautions for the build

- **Never invent Sanskrit, translations, notes, contributors or numbers.** Use bracketed placeholders (e.g. `[n] contributors`, `[price]`) and tell Jake what's missing.
- **Word-by-word data is the expensive part.** In Phase 3, start with editor-entered tokens for the Pratyabhijñāhṛdaya. Before drawing on external resources (e.g. the Digital Corpus of Sanskrit or the Sanskrit Heritage tools), check their licences with Jake.
- **Releases are immutable.** Fixes go into the next version; never edit a published release.
- **Credit everything.** Every accepted rendering and note shows its contributor in the reader, in the history, and in exports.
- **Keep editorial logic behind an API**, so the review queue can move to Tarka Desk later without a rewrite (open decision 4).
- **Jake doesn't apply patch files.** Make all changes directly in the repo, deploy them, and explain in plain language what changed and where to look.

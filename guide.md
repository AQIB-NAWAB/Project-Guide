# DevWeekends — Mentor's Guide to Authoring Project Guides

> **What this is.** A manual for mentors who write the project guides their mentees build from — by hand or with an AI doing the typing. The mentee ships a real, deployed app using your guide. They get requirements, the *why* behind every decision explained in depth, and reading woven in right where it deepens the work — never your solution code.
>
> **The first principle — don't empty the bucket on the desk.** A guide reveals the build *one step at a time*, in the order a working developer actually meets each problem. It reads like a story, not a spec sheet handed over all at once. Each chapter opens a small question, lays out the tempting-but-wrong approach and *why it falls short*, then explains the approach a professional reaches for and *why it wins* — and the mentee builds **that** one, understanding both. They never have to ship the broken version to learn why it's broken; the explanation, given in real depth, *is* the teaching.
>
> **It's a guide, not a brief — and not a textbook.** The mentee lives inside this for weeks, so it carries more than features. A working rhythm they're reminded of as they go. The occasional true story about how a real team solved the same problem, dropped in just to read. A nudge, here and there, to write up what they've learned under their own name. A brief lists what to build; a guide makes someone *want* to build it — and able to explain it afterwards. **A guide that teaches normalization and RBAC but never walks the mentee through folder structure, file creation, and a demoable screen at every phase has failed** — concepts serve the project (Rule 28), they don't replace it.
>
> **The test on every page of every guide you ship:** *Is the mentee doing the thinking, or is the guide doing it for them?* If the page thinks for them, it failed.

---

## How to use this manual

| You want to… | Read |
|--------------|------|
| Understand the philosophy and the rules | Part I |
| Know what you actually do (with or without AI) | Part II |
| Use AI to author a course (what it must ask you first) | §3 — Notes for an AI authoring a course |
| Write the mentee-facing guide | Part III |
| Lay the course out as phases, chapters, and sub-chapters | Part III §7 |
| Scaffold the actual project build (folders, files, steps) | §9 beat 6 "Build it", Rule 25, and §7.5 |
| Keep concept teaching from replacing project completion | Rule 28 and §7.6 |
| Write engaging, intuitive chapter stories | Part III §9 and Part IV |
| Scale the course and order it zero → advanced | Rules 19–21 and §15 (the complexity ladder) |
| Check a draft before publishing | Part VI |
| Look up a term in plain English | Appendix A |

Everything marked **mentor only** — the competency map, the personalisation menu, the viva rubric, the hint ladder — stays in this manual. It **never** appears on the mentee's guide.

---

# Part I — Foundations

## 1. Philosophy — escaping tutorial hell

**Tutorial hell** is finishing tutorial after tutorial and still not being able to build anything alone, because all you ever did was copy.

A good guide breaks that by changing **who decides** and by *explaining deeply* — **and by keeping the mentee's repo moving toward a finished product the whole way.** You supply the structure, the requirements, the reason each feature exists, and a full account of the choices hidden inside it — the tempting-but-wrong approach and exactly why it disappoints, the right approach and exactly why it's worth the effort. The mentee reads it, understands both sides, weighs them in their own words, and then builds the right version with their eyes open. **The understanding is the curriculum** — not a struggle through a broken build, but a real grasp of *why* the professional choice is the professional choice. **The deliverable is still the app** — concepts exist to make that app correct, secure, and fast; a course that only produces explanations has failed (Rule 28).

This is the deliberate change from "problem-first" teaching, where the mentee builds the naive version and feels it break. That wastes their time on code they'll throw away and teaches by frustration. Instead you *teach the contrast*: show the weak approach, dissect why it fails (with the concrete numbers — the query count, the bloated row, the leaked field), then show and justify the right one. They learn the same lesson, faster and without the detour, and they build only the version worth keeping.

This only works if you reveal the build the way it's actually lived — a step at a time. You don't open a feature by listing every requirement, every edge case, and the final design. You open with the one problem in front of them, explain it thoroughly, and let the next problem surface naturally from what they just built. A wall of requirements dumped at the top teaches nothing; a story that unfolds, explaining each decision the moment it arrives, teaches everything.

The strongest features don't stop at a single decision — they *climb*. The mentee solves a small thing, and solving it exposes the next thing to solve. For Example Registration needs somewhere to put the user's details; putting them there raises the question of where the *profile* goes; the profile leads to the password, the password to how the server remembers you between requests, and that to tokens, then refresh tokens, then verifying the account. Each rung is small, and each one earns the next. A mentee who starts at the bottom and keeps climbing stays gripped in a way no flat list of requirements can match.

A bad guide hands over working code, removes all the difficulty, or reads like a form — the same headings ("Guardrails", "Reflection", "Stretch") stamped onto every chapter. A bad guide is also *too thin*: a terse spec that names a topic, lists a few bullets, and moves on, explaining nothing — leaving the mentee to fend entirely for themselves. The mentee should experience **one continuous, professional, genuinely detailed story** per chapter — closer to a chapter of a good technical book than a worksheet.

You are writing a **structured product guide** — not a tutorial, and not a motivational essay.

## 2. The rules (non-negotiable)

Read these once and keep them in the back of your mind while you write.

| # | Rule |
|---|------|
| 1 | **Reveal step by step.** Open one problem at a time, in build order. Never dump the whole requirement list or the whole design on the mentee at once. |
| 2 | **Specs, not solutions.** Say what must happen and *why* — never the code they should type. |
| 3 | **Contrast, then build right — as a story.** For each decision: explain the obvious-but-wrong approach and exactly *why* it disappoints (with the concrete cost — query count, bloated row, leaked field), then the approach a professional uses and *why* it's better — and have the mentee build the right one. Teach the difference; **never make them implement the broken version to feel it.** Woven into paragraphs, not a ten-heading form. |
| 4 | **A chapter is a feature area; its sub-chapters follow the real build.** A **chapter** covers one coherent **feature or topic area** of the app — *Authentication*, *The product catalogue*, *Checkout* — and is broken into **20–30 granular sub-chapters** (Rule 24) that walk the actual build, one small step at a time. (A chapter can also be a pure *decision/concept* — "Why this database?" — which runs shorter.) The *chapter* is the milestone the mentee feels reaching; the *sub-chapter* is the step they complete in a sitting. A whole feature lives in **one** chapter, revealed sub-chapter by sub-chapter — never dumped at once (Rule 1). Group chapters into phases/parts (weeks). |
| 5 | **Descriptive chapter names.** Name each chapter after exactly what it covers — a concept ("Why MongoDB?"), a setup ("Setting up Mongoose"), a layer ("Backend: the auth API", "Frontend: the login screen"), a feature, or an improvement. Layer names like *backend*, *frontend*, and *database* are welcome when the chapter really is about that layer. The only bad name is a vague one ("Module 4", "Misc", "Stuff"). |
| 6 | **Prerequisites up front, pitched to the difficulty; reading woven after.** One baseline table at the start for what the whole project assumes — and set it to the course's level (§3): an advanced course that assumes CRUD must not baseline "what is HTTP." Never list a prerequisite the course itself teaches. Beyond that table, fold reading *into the prose* — a "worth reading about this" pointer or a short explainer right where a concept arrives, even mid-paragraph (see Rule 18). Rich and frequent is good; a rigid "read before you start" box stapled onto every chapter is not. |
| 7 | **Plain English.** Explain each piece of jargon the first time it appears, in one line, inside the prose. |
| 8 | **Motivate with why the skill matters** — never with "don't worry, this is hard." |
| 9 | **Scaffolding fades.** Early features are full stories; later ones shrink; the final bar-raiser is one line. |
| 10 | **No mentor-meta on the mentee's page.** No competency map, rubric, personalisation menu, template labels, or "as a mentor." |
| 11 | **Every chapter ends in a Definition of Done — or Key takeaways — and it's a gate.** Build/setup chapters end in observable, tickable outcomes you can demo or see; decision/concept chapters end in "you can now explain…" takeaways. The checklist **gates the next chapter**: the mentee doesn't move on until every box is ticked (see §8 and §11). One or the other, always. |
| 12 | **Short introduction.** A few paragraphs of orientation — why it matters, what they'll have, how the guide works, how they're judged. Not a chapter, not a pep-talk. |
| 13 | **AI is allowed; understanding is graded.** Mentees may build with AI. The viva checks they can explain their own work — the mentee guide stays clean of "AI Log" sections unless your program requires one. |
| 14 | **Scope by work, not clock.** Full-time pace (~50–60 hrs/week); progress is measured by chapters finished. |
| 15 | **Structured and readable — but not a form.** Clear headings, an overview table, and a prerequisites table are good. Repeating teaching-label subheadings on every chapter are not. |
| 16 | **Real links or topic names only.** Point to official docs or a real article — or just name the topic for them to search. Never invent a URL. |
| 17 | **Depth on every new concept.** When a chapter introduces something new, teach it as a full chapter would — the concept, the competing approaches, the trade-offs, the *why* — in real detail, so the mentee can *learn the topic* from it, not merely be told to go build it. (This doesn't fight Rule 9: later chapters shrink because they *reuse* concepts already taught, not because new ideas are left unexplained. Thin where it reuses is fine; thin where it introduces is a failed chapter.) |
| 18 | **Teach inline, link to deepen.** Don't gate the teaching behind "go read this." Explain each concept in the chapter, in plain words, *then* point to an article or doc for more — between paragraphs, right where curiosity peaks. The chapter teaches the topic; the links extend it, they don't replace it. |
| 19 | **A full course, not a pamphlet — fine-grained, never thin.** A substantial project is **~12–18 feature-chapters** (grouped into phases/weeks), and **each chapter breaks into 20–30 sub-chapters** (Rule 24) — because the build is revealed one small, guided step at a time and every concept is genuinely taught (Rule 17). The real "floor" is **granularity and completeness**, not a line count: *every meaningful build step gets its own sub-chapter* (create the folder · create the file · write this logic · wire it · test it), and *every new concept is taught in depth*. Never a chapter that races through a whole feature in three paragraphs, and never a "now build the rest" hand-wave that skips the steps. Don't collapse ten build steps into one fat sub-chapter to keep the count down; split them. Thin **or** coarse is a fail. |
| 20 | **Climb the complexity ladder, zero to advanced.** Start at the very bottom — project setup, config from the environment, validation — and climb step by step toward the harder concerns (TypeScript, rate limiting, async/background and batch processing, normalisation vs denormalisation, referencing vs embedding, indexing, caching…). **How far up you climb is set by the difficulty:** easy stops low, advanced goes high (see §3 and §15). |
| 21 | **Mandatory reads and hints between chapters.** At the points that matter, give the mentee *mandatory* articles to read so they build their own understanding, and short *hints* that nudge their thinking without handing over the answer. Both make the mentee think; neither solves it for them (see §10). |
| 22 | **Illustrate, don't just narrate.** Every teaching chapter must carry the **code blocks that make its ideas concrete** — data shapes and sample payloads, API contracts, schema/DDL and config sketches, the commands to run, side-by-side comparison snippets, an ERD or diagram — woven through the prose right where each idea lands. Don't ration them: a wall of prose with no blocks is harder to learn from and reads as thin. (This never means feature solution code — Rule 2; it means the *illustrative* blocks of §9 "The blocks a course is made of," used generously.) |
| 23 | **Every chapter moves the project — read and build, side by side.** A chapter is not an essay; it's a step in *building the actual app*. So every chapter must tell the mentee, concretely, **what to do on their project now** — *"create a `POST /products` route", "add this migration and run it", "send this bad request and watch it rejected"* — woven through the teaching, not bolted on at the end. The mentee should always feel the project **progressing in their own repo** as they read: learn a production practice, then immediately apply it to a real, named piece of the project, and see the result. **This includes concept chapters** — a "why" chapter still connects to the project through *applied* work on the real codebase (audit your own schema, run a query against your seeded data, stub the endpoint you're about to guard), never abstract reasoning alone. The test: after each chapter, can the mentee point at something in their repo that is new or changed because of it? If not, the chapter is teaching *at* them, not building *with* them. (Still specs, not solutions — Rule 2: tell them *which* route to create and *what* it must accept/reject, show the contract and the commands, but never hand over the handler's body.) And keep the thread **unbroken**: each build step opens by recapping where the project stands and names what it adds, so the mentee always knows the current state of their own codebase (continuity — Rule 27). |
| 24 | **Break each chapter into 20–30 sub-chapters — a guided arc, never a wall.** A chapter is *not* one long file; it's an ordered set of **20–30 sub-chapters** the mentee works through in turn, following the arc: *set the scene* (bold intro, motivation, a story/quote) → *learn it* (the options, the chosen approach, the *why*) → *reading* → *quiz* (reserved slot) → **build it in basic steps** (scaffold the folders/files, then implement step by step, each continuing the project) → *checklist* (the gate). Adapt the arc per chapter (a concept chapter runs shorter), but a chapter is **always** broken into small, navigable steps so the mentee always knows where they are and what's next. **This is a navigation skeleton, not a label-stamped form** (Rule 15): the *prose inside* each sub-chapter still flows as a story — don't stamp the same ten headings on every page. (Quizzes get a reserved sub-chapter slot but are **excluded for now** — see §7.) |
| 25 | **Scaffold the build — guide the hands, withhold the answer.** Explaining a topic is *not enough*; the mentee must be guided to *do it on the project*, concretely. Name the **exact folder and file to create**, the **table and column to add the index to**, the **component to build**, the **library to choose** (with the options weighed — *"you need to hold front-end state: Redux Toolkit vs Context — here's the trade-off"*). Sequence the actions (*"first the migration, run it; then the route; then the schema; then test with this request"*), give **hints** at the tricky parts, and tell them **how to verify** it worked. The *only* thing you never hand over is the **solution code** the mentee is there to write (Rule 2). "Teach the topic" was the old bar; "guide them to build it, step by step" is the bar. This is the single most important fix: the connection from *learning* to *a working project* must be explicit. |
| 26 | **Be intuitive and human — product-shaped, not a spec sheet.** Learners stay when a course feels like a person guiding them. Open chapters with motivation, a short real story, or a quote; keep the human side present throughout — never lose it. And name things the way a **product** would, not a system diagram: *"Create product"*, *"Admin dashboard"*, *"the vendor's shop"* — **not** "the product-creation endpoint" or "dashboard page component". Intuitive framing isn't decoration; it's what makes a hard topic approachable and a long course worth finishing. |
| 27 | **Guide the build in basic steps, and say what NOT to do.** The hands-on part of a chapter is not one "now build it" sub-chapter — it's **many tiny, sequenced sub-chapters**, each a single concrete action: *"create `src/modules/auth/`"* → *"create `auth.routes.ts` inside it"* → *"write the register handler — here's exactly what it must do"* → *"wire it into the app"* → *"test it with this request"*. Go **basic**: name the folder, the file, where it sits, what each must do — the mentee should never wonder *what to do next*, only *how to write the logic* (which they write themselves — Rule 2). At each step give explicit **✅ do this / ❌ don't do this** guidance — the mistakes to avoid teach as much as the steps to take. And every build sub-chapter **continues the project from its current state** (Rule 23): *"you now have working login; next we add refresh tokens — here's where they slot in."* This step-by-step, basic, do-and-don't guidance is the difference between a course that *explains* a project and one that *gets it built*. |
| 28 | **Concepts serve the project — never replace it.** A course that explains normalization, RBAC, and caching beautifully but never walks the mentee through folder structure, file creation, wiring, and a demoable feature at every stage has failed its primary job: **the mentee ships a working app.** Teach concepts in depth (Rule 17), but **between** build steps or **right after** the mentee has built the thing the concept names — not instead of building. No run of three or more chapters in a row that add no runnable piece of the product. A concept chapter still applies the idea to the real repo (Rule 23), but "write a matrix in `docs/`" alone is not enough if the app itself hasn't moved. The test at course level: at every phase boundary, can the mentee **run and demo** something new — a health check, a login screen, a vendor adding a product, a checkout — not just explain a textbook topic? |

---

# Part II — What you actually do

This is the part that trips people up, so it's deliberately plain. You can write a guide by hand or have an AI do the typing; either way the order is the same and the judgment is yours.

## 3. The whole job, step by step

**In short, as a mentor you do three things:** (1) pick the project and its difficulty, (2) pick the domain, then (3) use this manual to write the project guide, one chapter at a time. Everything below is those three things, plus the review that makes them good.

Do these in order:

1. **Pick the project and set its difficulty.** Choose a real app you want the mentee to build — a marketplace, an LMS, a social network, and so on. Then decide how hard it should be: **easy, medium, or advanced**. Keep it at that level the whole way through. The difficulty decides how much you explain, how big each chapter is, and **how far up the complexity ladder the course climbs** (§15). Roughly: an *easy* course tops out at clean setup, env config, validation, and correct CRUD; a *medium* one adds full auth/refresh, authorization and data isolation, pagination, and basic caching; an *advanced* one climbs further into rate limiting, async/background and batch processing, caching with invalidation, and normalisation-vs-denormalisation trade-offs.

2. **Pick the domain.** Choose the kind of business the app is for — a plant nursery, a B2B wholesaler, a music school. The domain must change *real decisions*, not just the name (see the test in §5). At the same time, choose the visual style and a few reading themes to go with it. These three together — domain, style, reading themes — are the **personalisation** that makes this mentee's guide unlike anyone else's. (See §5.)

3. **Write the competency map.** Before you write any feature, make a *private* list of the 6–12 things the mentee must be able to **build and explain** by the end. This is your checklist while you write and your question bank for the viva. Never show it to the mentee. (See §4.)

4. **Write the guide with this manual, one chapter at a time.** This is the main work — follow Part III for the shape of each chapter.

   If an AI is doing the typing, give it two things: *this manual*, and your brief (project, difficulty, domain, competency map, personalisation). Then hold it to the rules that matter most:
   - specs, not solution code;
   - explain the weak approach *and* the right one, then have the mentee build only the right one — never make them build the broken version;
   - deep chapters that actually teach the concept;
   - reading woven through the text, with **"Topics to read"** and **"Interesting to read"** blocks where they help (see §10);
   - **each chapter is a feature area broken into 20–30 sub-chapters** following the arc — set the scene → learn it → reading → build it (in basic steps) → checklist — not one undivided wall (Rules 4, 24, §7);
   - **a Build it cluster of many tiny steps** that scaffolds the project — name the folders/files to create, the table/column, the library to choose; one step per sub-chapter; ✅do/❌don't at each; continuity ("you have X, now add Y"); how to verify — all without handing over the solution code (Rules 25, 27, §9.6). This is the part most drafts get wrong: they teach but don't *guide the build*;
   - **intuitive, human, product-shaped** framing and names throughout (Rule 26);
   - **project skeleton early**, then **backend/frontend paired** per feature — concepts never replace build steps (Rules 28, §7.5–7.6);
   - a Definition of Done (or Key takeaways) at the end of every chapter.

   You don't need a clever prompt. Point the AI at this manual and hold the line — but **first have it ask you the questions below** so it isn't guessing the difficulty, stack, or domain.

5. **Review it like a hostile reader.** Read the draft and look for any place where the guide does the thinking *for* the mentee, or where a chapter is too thin. Fix one chapter at a time; don't regenerate the whole guide to fix one part. (Common problems and fixes: §6.)

6. **Build the project yourself** — or have someone else build it from the guide alone. Nothing else reveals a missing step or the wrong difficulty as fast.

7. **Get reviewers and ship.** Three or four readers, some senior, some junior. Only ship once the smell tests in §20 pass.

Remember: the AI is fast at writing, but *you* own the judgment — the difficulty, the domain, where the hard parts are, and whether the mentee is really doing the thinking.

### Notes for an AI authoring a course *(ask the mentor first)*

If you are an AI helping a mentor write a course from this manual, **do not start writing chapters until you have interviewed the mentor.** A course is only as good as the brief behind it, and the mentor holds that brief. Ask this short round of questions first, present the options clearly, and wait for answers — never silently assume:

1. **Difficulty level — and explain the options so the choice is informed.** Ask: *easy, medium, or advanced?* Then describe what each means here, so the mentor isn't guessing:
   - **Easy** — for someone newer. Teaches the fundamentals and tops out at clean project setup, configuration, validation, correct CRUD, and basic auth. Fewer chapters; more explanation per step.
   - **Medium** — assumes the mentee can already build CRUD. Climbs into full auth (hashing, access + refresh tokens), authorization and data isolation, pagination, error handling, and basic caching.
   - **Advanced** — assumes all of the above as a given. Climbs further into rate limiting, async/background and batch processing, caching with invalidation, normalisation vs denormalisation, indexing strategy, and idempotency.

   The level sets how far up the complexity ladder the course climbs and how many chapters it runs to (see §3 step 1, §15, and Rule 20). Offer a recommendation if the mentor is unsure.

2. **Tech stack.** Ask what they want to build with — language, framework, database, key libraries — or propose a sensible default for the chosen difficulty and ask them to confirm. The stack makes chapters concrete: you can't write "set up Mongoose" without knowing it's Mongo + Node, or "the job queue & worker" without knowing the queue library.

3. **Project and domain.** Ask what app it is and what *domain* it serves (the kind of business). The domain must change real product decisions, not just the title (§5) — offer two or three concrete suggestions if they're unsure.

4. **Scope and timebox.** Ask the duration and weekly hours (e.g. "4 weeks at ~60 hrs/week"). This sizes the course — how many feature-chapters and phases, how many sub-chapters each runs to, and how far up the ladder you go (§15).

5. **What the mentee already knows.** Ask what to assume, so the prerequisites are pitched to the level and never re-teach the basics (§8).

Once you have the answers: write the **competency map (§4)** for yourself, then propose the **course outline grouped into phases → feature-chapters** (mindset → project → the build) and **confirm it with the mentor before writing any chapter**. The outline must include an early **project skeleton chapter** (§7.5) and **paired backend/frontend chapters** per feature (§7.6) — not a long stretch of concept-only chapters before the repo runs. Then write the **`01-introduction/` chapter first**, and proceed **chapter by chapter, pausing for review** — each chapter broken into its 20–30 sub-chapters (§7), with a real **Build it** cluster of basic steps (§9.6) that is the **largest block** in feature chapters. If the mentor hasn't told you the difficulty, stack, or domain — *ask*, don't invent.

## 4. The competency map *(mentor only — write it first)*

The competency map is the backbone of the whole project. It is a private list of what the mentee must be able to **do and explain** by the end.

**How to write it:** before you write any feature, list 6–12 skills. Each skill must later appear in a feature (so the mentee learns it) *and* in the viva (so you can test it).

Example, for a multi-vendor marketplace:

- Model vendors, customers, stores, and products with correct ownership
- Hash passwords and issue role-aware tokens; explain the threat model
- Enforce that one vendor cannot touch another vendor's data
- Build a multi-vendor cart and split checkout into per-vendor orders with price snapshots
- Avoid N+1 queries on the catalogue; explain joins and indexes
- Store images in object storage, not in a database row
- Cache catalogue reads and invalidate them when products change
- Deploy with secrets in environment variables, not in Git

**How to use it:** keep this list next to you while you write, and tick each item off as a feature covers it. It is a checklist for *you* — never publish it in the mentee's guide.

## 5. Personalisation *(mentor only)*

Two mentees building the same project should never be able to hand in the same code. Personalisation prevents that, and it also makes each mentee feel the product is truly *theirs*. You assign **three things** to each mentee, and you keep the full menu (below) to yourself.

**1. Domain — the most important one.** This is the kind of business the app is for, and it must *change real decisions*. For example:

- a plant nursery needs care instructions and a watering schedule on each listing;
- a digital-downloads shop needs license keys and file delivery instead of shipping;
- a B2B wholesaler needs tiered pricing and minimum order quantities.

Test your choice like this: if swapping the domain wouldn't change the database or the features, you've only changed the name — pick a different domain.

**2. Design language.** The visual style — minimal editorial, bold maximalist, retro terminal, Swiss grid. This simply sets how the UI should *feel* when you describe features. It is not a lecture on design systems.

**3. Reading themes.** Across the whole project, mix up the *kind* of articles you point to: a deep dive on one concept, a real post-mortem story, an X-vs-Y comparison, an opinionated best-practices piece, and at least one topic slightly outside the syllabus to stretch their curiosity. Pick about five for the whole project.

**How it reaches the mentee:** as *one natural sentence*, for example — *"You're building for a local plant nursery in a clean editorial style — name the product yourself."* Never write the word "personalisation," never show the menu, and never tell the mentee that anything was assigned.

The menu to choose from (yours only):

| Choose one | Examples |
|---|---|
| **Domain** | handmade crafts · local grocery · digital downloads · plant nursery · B2B wholesale · events / ticketing |
| **Design language** | minimal editorial · bold maximalist · retro terminal · Swiss grid |
| **Reading themes (≈5 total)** | concept deep-dive · post-mortem · X vs Y · opinionated best-practice · one topic outside the syllabus |

## 6. Fixing a weak draft

Most first drafts — whether you or an AI wrote them — fail in the same few ways. Your job in review is to find the moment the guide stops making the mentee think, and fix it. Use the table below: spot the problem on the left, apply the fix on the right.

| If you see this problem… | Do this to fix it |
|---|---|
| Reassurance ("don't worry, this is hard") | Cut it. One confident line on why the skill matters. |
| Any code the mentee should write | Replace with behaviour + Definition of Done + what to read. |
| A chapter that is only a label with no real content (e.g. an empty "Backend" heading) | Make every chapter a full, detailed story — concept/why → build → Done (or Key takeaways). The layer *name* is fine; the emptiness is not. |
| The mentee is told to build the broken version "to feel the pain" | Remove it. Explain why the weak approach is bad (with the real cost) and what the right approach is, then have them build only the right one. |
| A chapter that is just a short spec or a few bullets | Expand it into a full chapter that explains the concept, the approaches, and the trade-offs, with reading woven in (Rule 17). |
| A concept is named but not explained — just "go read this" | Explain the concept yourself first, then add the link to go deeper (Rule 18). |
| The same subheadings on every chapter (Guardrails, Reflection, Stretch…) | Merge into one story; keep only the Definition of Done (or Key takeaways) as a list. |
| "Read before you start" on every chapter | Keep it only where that reading is genuinely required *right then*. |
| Invented URLs | Use a real link or just the topic name. |
| A vague outline with no real chapters | Restructure into **phases → feature-chapters → 20–30 sub-chapters**, each a guided step (§7, §9). |
| **The topic is taught but the mentee isn't guided to *build* it** (no folder/file named, no "do this on your project") | Add the **Build it** cluster (§9.6, Rules 25, 27): name the exact files/folders, the table/column, the library to choose, the steps in order, and how to verify — withhold only the solution code. |
| **The build is one big "now build it" step**, not broken into basic steps | Split it into **many tiny sub-chapters** (Rule 27): scaffold the files → write this → wire it → test it. One concrete action per step. |
| **No "what not to do" guidance** — only the happy path | Add **✅ do / ❌ don't** callouts at the build steps (Rule 27): the trap to avoid teaches as much as the path to take. |
| **Steps don't connect** — each reads as if the project started fresh | Open each build step with a **continuity recap** (Rules 23, 27): "you now have X; next we add Y, here's where it slots in." |
| **An undivided wall of prose** — one long file, no sub-chapters | Break it into the **20–30 sub-chapter arc** (§7, Rule 24): set the scene → learn it → build it (basic steps) → checklist. Structured, not open. |
| **A "chapter" is one tiny topic** (e.g. "password hashing" alone) | A chapter is a **whole feature area** (Authentication); that topic is a *sub-chapter* inside it (Rule 4). |
| **Dry, technical names** ("dashboard page component", "the product POST handler") | Rename to be **intuitive and product-shaped** (Rule 26): "Admin dashboard", "Create product". |
| **Cold, all-business chapters with no human side** | Open with motivation, a real story, or a quote; keep the human voice present (Rule 26). |
| **Concept chapters stacked back-to-back** — normalization, then RBAC, then threat model, with nothing runnable added | Reorder: **project skeleton first** (§7.5), then build; teach each concept **adjacent to** the feature it names (Rule 28). Cap concept-only chapters at **one in a row** unless the next chapter immediately implements what the concept introduced. |
| **Backend built, frontend missing for half the course** | **Pair server and client chapters** per feature (§7.6). Every phase ends with something **demoable in the browser**, not only in Postman. |
| **Concept chapter ends with only a doc or learning log** — no change to the running app | Add applied repo work that **moves the product**: stub the route, run the migration, seed and query, wire a placeholder screen — something the mentee can run (Rule 23, 28). |

Work through it one chapter at a time: fix a chapter, re-read it to check the fix, then move on to the next.

---

# Part III — The shape of a mentee course

Before the skeleton, the idea behind it: the mentee meets the project the way a developer meets a new codebase — orientation first, then one problem at a time, explained in depth, with reading woven through the prose wherever it deepens the work. The order below exists to make that unfolding feel natural. Keep the headings visible so the mentee can always see where they are.

## 7. The course as a structured set of files

A guide isn't one long page — it's a **course**, the way an Educative track or a good textbook is: one folder per project, organised into **phases**, then **chapters**, and each chapter into small **sub-chapters**. That structure is the point. An open, vague wall of prose is hard to learn from and easy to abandon; a clearly structured course — where the mentee always knows *where they are and what to do next* — is what keeps them moving. Structure first, then fill it with the teaching.

### The three phases of a course

Every course moves through three phases, in order — name them in the outline so the mentee feels the arc:

1. **Mindset** — get the mentee's head right: what they're going to build, why it's worth building, how to think about it, and how the course works. (This is the intro chapter, §8.)
2. **The project** — what exactly they're building: the product, the people who use it, what's in and out of scope. (In the intro, and re-grounded as each feature arrives.)
3. **The build** — the feature-chapters themselves, grouped into **phases (weeks)**, climbing the complexity ladder (§15) one feature at a time.

*First I get my head right, then I understand the thing, then I build it rung by rung* — the mentee should be able to feel that shape.

### Chapters (feature areas), and the sub-chapters inside them

A **chapter** covers one coherent **feature or topic area** of the app — *Authentication*, *The product catalogue*, *Checkout*, *Background jobs* — and a substantial project has only **~12–18 of them**, grouped into phases/weeks (Rule 4). The chapter is the **milestone** the mentee feels themselves reach ("I built auth").

Each chapter is then broken into **20–30 sub-chapters** (Rule 24) — the small, guided **steps** the mentee actually completes one at a time. This is the heart of the new structure: a feature isn't explained in one long page, it's *walked*, step by basic step, so the mentee always knows exactly where they are and what to do next. The sub-chapters follow an **arc** (clusters, not a rigid form):

| Cluster (≈ how many) | The sub-chapters in it |
|---|---|
| **Open & motivate (≈3)** | *Set the scene* — a **bold intro**: what this feature is and what it means · *Why it matters* — motivation, a short **real story or quote**, the stakes · *What you'll build & the goals* for this chapter. |
| **Learn it (≈4–6)** | *The approaches* — the options on the table (e.g. sessions vs tokens) · *The decision* — which one this project uses, with **pros/cons and a clear mandate** (§9) · *Deep-dive* — what the chosen thing actually *is* (e.g. what a JWT holds and how signing works), taught in real depth (Rule 17) · *Prime your thinking* — the **mandatory reads** (§10). |
| **Check (≈1)** | *Quiz* — **reserved, excluded for now** (placeholder; see the note below). |
| **Build it in basic steps (≈10–14)** | *Where the project is now* — a continuity recap + what we add · *Scaffold* — **create these folders and files; here's how the structure looks** · then **one sub-chapter per build step** (create this file → write this logic → wire it → test it…), each: continues from the prior step (Rule 23), is **spec-not-code** (Rule 2), names exactly *what to do next* (Rule 27), carries **✅ do / ❌ don't** guidance and a **hint**, and ends with **how to verify it works**. |
| **Wrap (≈2–3)** | *Practice / stretch* (optional) · *Recap & what's next* · *Checklist & learning log* — the **Definition of Done** (or **Key takeaways**) **gate** (§11). |

That arc naturally yields **20–30 sub-chapters** for a real feature; a pure **concept chapter** (e.g. "Why this database?") runs shorter on *Build it* but still includes **applied project work** (Rule 23) — never "read only." **The Build it cluster must be the largest block** in every chapter that ships a feature: roughly half or more of the sub-chapters. If *Learn it* dominates and *Build it* is one page, the chapter will teach a topic without completing the product (Rule 28). **The arc is a navigation skeleton, not a label-stamped form** (Rule 15): the *prose inside* each sub-chapter still flows as a story, and you don't print the cluster names as headings on the mentee's page — they're how *you* structure the chapter, not a worksheet you stamp on them.

> **A note on quizzes (excluded for now).** Each chapter (the *Check* cluster) and each phase has a natural place for quizzes/assessments. **Leave them out for now** — they belong to the future interactive platform where learners submit answers and run code. Leave the quiz sub-chapter as an empty placeholder so the structure is visible, or omit it. Until quizzes exist, the **learning-log questions** (§11) carry the "make them think" load.

### Folder layout

A **chapter is a folder** (a feature area), and its **20–30 sub-chapters are ordered files** inside it — here, the *Authentication* chapter:

```
project-name/                          ← one folder per project (the course)
  01-introduction/                     ← Phase 1: mindset + the project (the orientation chapter, §8)
    01.01-what-youre-building.md
    01.02-how-to-use-this-course.md
    01.03-the-product-and-its-people.md
    01.04-scope.md
    01.05-prerequisites.md
    01.06-course-outline.md
  phase-2-identity/                     ← a phase/week groups feature-chapters
    07-authentication/                  ← a CHAPTER = one feature area = a folder
      07.01-set-the-scene.md            ┐ open & motivate
      07.02-why-it-matters.md           │   (bold intro, a real breach story, the stakes)
      07.03-what-youll-build.md         ┘
      07.04-the-approaches.md           ┐ learn it
      07.05-the-decision.md             │   (sessions vs tokens → which we use & why)
      07.06-what-a-jwt-is.md            │
      07.07-prime-your-thinking.md      ┘   (mandatory reads)
      07.08-quiz.md                     ← reserved, excluded for now
      07.09-where-the-project-is-now.md ┐ build it, in basic steps
      07.10-scaffold-the-auth-folder.md │   (create these folders/files)
      07.11-hash-passwords.md           │
      07.12-the-register-endpoint.md    │   one sub-chapter per step, each continuing
      07.13-the-login-endpoint.md       │   the project, with ✅do/❌don't + a hint + verify
      07.14-issue-an-access-token.md    │
      07.15-add-refresh-tokens.md       │
      07.16-verify-the-email.md         │
      07.17-the-auth-middleware.md      ┘
      07.18-practice.md                 ┐ wrap
      07.19-recap-and-whats-next.md     │
      07.20-checklist.md                ┘   (the gate)
    08-authorization/
      ...
  ...
  99-closing/
    99.1-ship-it.md
    99.2-document-it.md
    99.3-bar-raiser-and-evaluation.md
```

The first chapter is always **`01-introduction/`** (the orientation — *not* a `README.md`). Number chapters so they sort in build order, and sub-chapters `NN.01`, `NN.02`… within each. Group chapters into **phase/week folders**. Name everything **intuitively** (Rules 5, 26): the chapter folder is `checkout`, not `order-service`; a build sub-chapter is `add-to-cart`, not `cart-controller`; `admin-dashboard`, not `dashboard-page-component`.

(For a *very small* course you may keep a chapter as a single file with the sub-chapters as headings inside it. For a real project, the **folder-of-sub-chapters** structure is the default — it's what keeps a long course navigable and stops it becoming a vague wall.)

### What maps to what

| Course term | Here | Lives in |
|---|---|---|
| Course | The project | The top folder |
| Phase / part / week | A group of feature-chapters | A subfolder |
| **Chapter** | One **feature area** (Authentication, Checkout…) — the milestone | A chapter **folder** |
| **Sub-chapter** | One **guided step** — set the scene, learn it, *create this file*, test it, checklist… | A `.md` file inside the chapter folder |
| Intro chapter | Mindset + the project + how to use the course + prerequisites + outline | `01-introduction/` |

### The order the mentee reads in

```
01-introduction/        → Phase 1: mindset + the project + how the course works
phases of chapters/     → the build, one feature-chapter at a time; each chapter runs its arc:
   open & motivate  (set the scene → why it matters → goals)
   → learn it       (the approaches → the decision → deep-dive → reading)
   → [quiz slot]
   → build it       (where we are now → scaffold the folders/files → step → step → step…)
   → wrap           (practice → recap → checklist, the gate)
99-closing/             → ship, document, prove it
```

Each chapter's final **checklist gates the next chapter** (§11); each phase is a milestone the mentee can feel themselves reach.

### The mandatory project skeleton (early — before deep concepts)

The most common way a course fails is also the easiest to prevent: **the mentee reads about RBAC and normalization for a week while their repo is still empty.** Fix that with a dedicated early chapter — usually the first or second *build* chapter after the intro — whose only job is to **stand the project up on disk** so every later chapter has somewhere concrete to attach.

This chapter is not optional for a full-stack project. It must cover, in order:

1. **What you're creating** — one repo (or two, if you mandate separate repos), with **`server/`** and **`client/`** (or your stack's equivalent) named plainly. Show the **tree** before any logic: the mentee should *see* the shape of the app before filling it in.
2. **Why the structure matters** — compare the two folder philosophies (layer-based vs feature/module-based) with pros and cons, **mandate one for the course** (§9), and say why changing it later breaks every subsequent chapter.
3. **Server-side scaffold** — create the backend entry files (`server.ts`, `app.ts`, config, shared `db` client), wire a health route, run it, verify `200`. Name every folder and file; explain what each *will* hold, even if it's empty today.
4. **Client-side scaffold** — create the frontend app (Vite/CRA/your choice), set the **folder convention** (e.g. `src/pages/`, `src/components/`, `src/api/`, `src/store/`), add routing shell and a placeholder home page, run it, verify it loads.
5. **Client vs server — taught here, not in the abstract** — one short, concrete explanation woven into the scaffold: the browser runs the client; the server runs the API; they talk over HTTP; env vars live on the server; the client never holds secrets. Point at *their* two folders while explaining — not a generic diagram alone.
6. **Connect them** — configure the client's API base URL, call `GET /health` from the client (even a one-line `fetch` in `useEffect`), show the response in the UI. **That's the first end-to-end demo** — and it gates everything after.

Each of the above is its own sub-chapter (Rule 27): create the folder → create the file → explain why → wire → verify. The mentee finishes this chapter with **both apps running**, a visible folder tree they can navigate, and a checklist that proves it.

> **✅ Do:** show the full tree after each scaffold step so the repo always has a readable shape. **❌ Don't:** skip the client until "Week 3" — a backend-only course for weeks teaches concepts *about* a product the mentee has never seen run in a browser.

### The completion thread — backend and frontend stay paired

A marketplace (or any full-stack app) is not complete until **both** sides exist. The course outline must make that explicit:

- **Interleave, don't orphan.** When a feature has an API and a UI (auth, vendor dashboard, catalogue, cart, checkout), place the **client chapter immediately after** the **server chapter** for the same feature — or split one chapter into backend build steps then frontend build steps. Don't stack five backend or concept chapters while the frontend still shows a blank page.
- **Demoable at every phase.** At the end of each week/phase, state what the mentee can **run and show**: Week 1 ends with health check + folder tree; after auth, a login screen that returns a token; after products, a vendor form that creates a row in the database; after checkout, an order in the UI *and* in the database. If the phase ends in "you can explain normalization," the phase failed.
- **Concept chapters come after the build they explain.** Teach normalization *after* the mentee has built normalized tables and hit a snapshot decision — not as a three-page essay before they've created `products`. Teach RBAC *immediately before* the middleware chapter that implements it — not three chapters before with only a `docs/` matrix. The concept earns its place by naming something they already built or are about to build *in the next sub-chapter*.

The test for the whole course: **could someone clone the mentee's repo at the final checklist and use the app?** If the answer is "they could pass a viva but the checkout page doesn't exist," the guide taught topics, not a project.

## 8. The opening sections (the first chapter, `01-introduction/`)

The course opens with an introduction chapter — the folder **`01-introduction/`** (Phase 1: *mindset + the project*). It orients the mentee before they build, and it does one job most guides skip: it tells them **how to use the course and how progress is gated**. Its parts below map to the intro's sub-chapter files (`01.1-…`, `01.2-…`); on a small course they can be headings in one file, but the order and the arc stay the same.

**Format matters here.** Give each part its **own heading on its own line** — never bury a heading as bold text inside a paragraph. A reader should be able to skim the headings alone and understand the shape of the page. Keep each part tight, but structured.

Order the intro chapter in this arc — *what → how → who → what you need → the plan*:

### 1. Introduction — what you'll build and why it matters

A few short paragraphs: why this project is a real career skill, and one sentence on what they'll have at the end. No fear-mongering, no pep-talk.

### 2. How to use this course

This is the section that makes a course *feel* like a course, and it's usually missing. Explain, plainly and engagingly:

- The course is a series of **chapters**, worked through **in order**.
- Each chapter **teaches** — it explains the *why*, compares approaches, and points to **mandatory reads** and **hints** as you go.
- **Every chapter ends in a checklist** — its Definition of Done (or Key takeaways).
- **The checklist is a gate.** The mentee does **not** move to the next chapter until every box is ticked. Say it plainly: the checklist is your permission to proceed, and the thing that keeps you honest. (See §11.)
- **You keep a learning log.** The mentee keeps a **`learning-log/`** folder in their repo and writes their answers to each chapter's questions in a file there (`learning-log/NN-chapter-name.md`). It's **mandatory** — it's the evidence the gate was really cleared, the rehearsal for the viva, and what the mentor or an AI reviewer reads. (See §11.)
- **How you're judged:** on understanding, at a final viva — the per-chapter checklists and your learning log are how you stay on track for it.
- **When you're stuck:** read the error and the docs, try a smaller test, then ask with "here's what I expected vs what happened."

### 3. The project, then the actors

Describe the **product** first — a short narrative so the mentee understands the *thing* before meeting the people in it. *Then* the **actors** (vendor, customer, admin if any): one short paragraph each on what they need. Understand the system, then the people — not the other way round.

### 4. In scope / Out of scope

The major functional requirements as a clear list, then the deliberate exclusions (payments, an admin panel, event-driven architecture) so the mentee doesn't gold-plate the wrong things. Optionally a one-line user flow: *"Customer browses → adds to cart → checks out → vendor sees the order."*

### 5. Before you start — the prerequisites table

One table, before the first real chapter. Two rules govern it: keep it **relevant to this project**, and keep it the **necessary minimum** — the floor the course stands *on*, deliberately **easier than the course itself**. Prerequisites are not a gauntlet to scare people off; they're the few things a mentee must already be comfortable with to begin.

- **Pitch it to the difficulty (§3) — but no higher.** A medium course that assumes CRUD shouldn't baseline "what is HTTP", but it also shouldn't demand mastery of everything the course teaches. List the comfortable basics, not the hard parts.
- **Never list a prerequisite the course itself teaches.** If an early chapter will teach it, it's not a prerequisite — move it into the chapter. The table is the floor; the chapters are the climb.

Example baseline for a **medium** project that assumes CRUD — relevant, necessary, and easier than the course:

| You should be comfortable with | Why you need it |
|---|---|
| Building a basic CRUD REST API on a framework | The course starts here and climbs |
| SQL basics — tables, joins, foreign keys | The data model and queries build on it |
| async / await in your language | Background jobs (later) use it |
| Everyday Git — commit, branch, `.gitignore` | Daily commits; secrets stay out of history |

Mark rows **must have** or **revise if rusty**. You can add a one-line "*helpful but taught anyway*" note (e.g. Docker, a typed language) so no one is scared off. Use **topic names** when you don't have an official link; never invent a URL.

### 6. The course outline

The mentee's roadmap and table of contents — the **feature-chapters** in order, grouped by **phase/week** (Rule 19). A "4 weeks at ~60 hrs/week" course reads as four week-blocks, each holding a few feature-chapters; you can show each chapter's sub-chapters as a nested list, or just give each chapter a one-line "what you'll build" and let its folder reveal the steps (see §15 on timeboxing). Give each chapter a short "Done when…".

| # | Chapter | Done when… |
|---|---------|-----------|
| 2 | [Project skeleton — server, client, first connection](02-project-skeleton.md) | Both apps run; folder tree in place; client calls `GET /health` and shows the result |
| 3 | [Why a relational database?](03-why-relational.md) | Can explain why relational fits this project, and the trade-offs |
| 4 | [Project setup & config](04-project-setup-and-config.md) | Boots from validated env config; no secret in Git |
| … | … | … |

(The intro is chapter 1, so the build chapters start at 2.) Keep the app demoable as you go: when a feature spans a backend and a frontend chapter, place them next to each other.

### 7. Your working rhythm

Close the intro with a short list of habits to keep from Day 1 — commit daily, secrets out of Git, read the error before asking. You'll re-surface these lightly through the course (§15), not repeat them as a box on every chapter.

## 9. How to write a chapter (the story pattern)

Each **chapter** is read start to finish, like a chapter of a technical book — and it should be *substantial*. Thin chapters are the most common failure (Rule 17). The teaching beats — the contrast, the *why*, the concepts, the reading — are **woven into the prose**, not pasted in as a template with ten labels.

**Chapters come in a few types, and the beats flex to fit (§7):**

- A **decision / concept chapter** ("Why MongoDB?", "What is Mongoose?") is mostly beats 1, 3, 4, 5 — context, the options compared, the concept taught in depth, reading. It ends in **Key takeaways** instead of a Definition of Done. It usually has no *new feature* to ship — but it must still **connect to the project** (Rule 23) and **must not block project progress** (Rule 28): give the mentee concrete applied work on their *real* codebase — audit their own schema against the concept, run a query against their seeded data to *see* the trade-off, stub the route the *next* chapter implements, or add the migration the next chapter runs. A concept chapter teaches a "why," but the mentee should still *do* something runnable or repo-visible with it — not just read and write a learning log. **Schedule concept chapters after the build they explain, or immediately before the build that implements them** — never as a long runway of essays while the app stands still (§7.6).
- A **setup or build chapter** ("Set up Mongoose", "Backend: the auth API") uses the full flow and ends in a **Definition of Done**.
- A **frontend/layer chapter** focuses on its slice (the screen, the API) and ends in a Definition of Done for that slice.

And remember the climb (§1): the best chapters aren't one point and done, they're a *ladder*. Each step the mentee finishes should surface the next question, so they keep ascending instead of reading a flat list. Use the beats below to build a single rung — then let the chapter run through several of them.

The flow below is a shape to adapt, not a form to fill:

1. **Context.** Where the project is now and what this chapter adds. Paragraphs — not a "Where you are" label.

2. **The point of the chapter.** What this chapter is for, in plain language — the decision to make ("which database?"), the thing to set up ("connect Mongoose"), or what the feature must do and how it should *feel* ("a vendor sees only their own listings, like a shop back office"). Drop a minimal UI hint only when it helps. No button-by-button instructions.

3. **Compare the two approaches — explain them, don't make the mentee fail.**
   As a mentor, put two options side by side. First, name the approach a beginner would naturally reach for, and explain *why it's a bad idea* — in plain words, with one real number. For example: "this runs one database query for *every* row in the list," or "this copies the same name onto a thousand records." Then show the approach a professional would choose, and say why it's better.
   The important word is **explain**. You are *not* telling the mentee to build the bad version and watch it break. You describe the cost — show a query log, a payload size, or what the database would look like — so they understand the problem without writing code they'll throw away. They read both options, think it through, and build the good one.

   **When the two options are both legitimate** — two project structures, two libraries, not a wrong-vs-right — don't just lay them out and walk away. Give the **pros and cons of each** honestly (a small comparison table is ideal), *then* **make a clear, mandatory recommendation** for the course. A foundational choice the rest of the course builds on (folder structure, the ORM, the queue library) must be *decided*, not left open: if two mentees choose differently, later chapters stop lining up — "add this to `modules/products/`" won't match a layer-based repo — and an undecided mentee just stalls. So weigh them fairly, pick one, say plainly that it's the course's required path and why, and move on.

4. **Teach the concept properly — enough to actually learn it.**
   As a mentor, don't just name a tool and move on. If the feature uses a JWT, explain what a JWT actually *is*: what it holds, what "signing" it does, and what it costs you. Define every new word the first time you use it, in plain English.
   A simple test: after reading your chapter, could the mentee explain the idea to a friend? If all they'd have is a working endpoint they can't explain, you haven't taught it yet — go deeper.

5. **Weave the reading through the chapter — don't dump it in one block at the end.**
   As a mentor, every time you introduce a new idea, do two things: explain it in your own words, then point the mentee to one good article or doc to go deeper. Put that pointer right next to the idea — even mid-paragraph: *"This is the classic stateless-vs-stateful trade-off; it's worth reading a short piece on it before you choose."*
   You can do this often — a chapter full of "here's the idea, and here's where to learn more" reads like a good book. The one rule: **explain first, link second.** Never replace your own explanation with "go read this."
   When a moment needs more than an inline pointer, use one of the named blocks from §10: **"Mandatory reads"** (must read before the next chapter), **"Topics to read"** (recommended deeper dives), or **"Interesting to read"** (a true story, just to enjoy) — plus a **Hint** where a step is genuinely tricky. Keep them for when they're genuinely needed; the plain inline explanations can be frequent.

6. **Build it — scaffold the project across many basic steps (the beat most guides skip, and the most important one).**
   As a mentor, once you've taught the topic, **guide the mentee to actually build it on their project** — concretely, in order, without handing over the solution code (Rules 2, 25, 27). This is not *one* "now build it" sub-chapter; it's the whole **Build it cluster** (§7) — **many tiny sub-chapters**, each a single concrete step. This is exactly where weak guides go silent: they explain a concept beautifully and leave the mentee staring at their editor not knowing what to touch. Don't do that. Be specific, and go **basic**:
   - **Start by scaffolding — show the structure.** The first build sub-chapter creates the folders and files: *"create `src/modules/auth/`, and inside it `auth.routes.ts`, `auth.service.ts`, `auth.schema.ts` — here's how the tree looks now."* The mentee should *see* the shape before filling it in.
   - **One step per sub-chapter.** Don't cram the whole feature into one page. *Create the file* is a step. *Write the register handler's logic* is the next. *Wire it into the app* is the next. *Test it* is the next. Each small, each finishable, each its own sub-chapter (Rule 27).
   - **Name the exact thing.** Not "add validation" but *"create `product.schema.ts` and define the create-product body shape."* Teaching indexing? *"add an index to `products(status, created_at)` — the columns your catalogue filters and sorts on."* Front-end state? *"you now need to hold the cart in state — choose a state library; weigh Redux Toolkit vs Context (here's the trade-off) — and put the cart slice in `src/store/cart.ts`."* Name every basic detail a real builder hits.
   - **Carry the project forward (continuity).** Open each build step by recapping where the project stands: *"you now have working login that returns a token; it's not checked on any route yet — let's fix that. Create…"* The mentee never loses the thread of their own codebase (Rule 23).
   - **Give ✅ do / ❌ don't at each step.** The pitfalls are as valuable as the steps: *"✅ derive the user's id from the verified token. ❌ don't trust an id in the request body — that's how one user edits another's data."* Name the trap, not just the path.
   - **Give the contract, the command, and a hint — not the body.** Show the route signature, the request/response shape, the CLI command, the table DDL sketch (Rule 22), and a hint at the tricky part (*"the check must run before the handler — think about middleware order"*). Then let them write the function.
   - **Tell them how to see it work.** End every step with the concrete verification — the `curl`, the SQL query, the screen state — so they *know* it's done before moving to the next step.

   The test for this cluster: **could a mentee, working through these sub-chapters, build the whole feature — always knowing the next file to create, what it must do, what to avoid, and how to verify — without ever being shown the answer?** If they'd be left guessing what to do next, the steps are too coarse; if you've written the function for them, it's too much. *Guide the hands, withhold the answer.* (This applies to frontend as much as backend: "build the admin dashboard" is never one step — scaffold the page/component files, set up routing, decide where state lives and which library holds it, wire the API calls, handle the loading and error states — each its own small step.)

7. **Close the chapter — Definition of Done, or Key takeaways.**
   As a mentor, end the chapter cleanly:
   - For a **build or setup chapter**, describe the behaviour you expect in clear sentences — what it should *do*, never how to code it — then add the **Definition of Done**: a short checklist of outcomes you can see or demo.
   - For a **decision or concept chapter** (one with nothing to build, like "Why MongoDB?"), there's nothing to tick off, so end with **Key takeaways** instead: a few bullet points stating what the mentee should now be able to *explain*.
   Every chapter ends in one or the other — never both, never neither.

8. **An optional harder path.** One line at the end for strong mentees, if it fits.

**Never** stamp these onto the mentee's page as repeating headings: Guardrails, Check yourself, Reflection, Stretch, Hint ladder, Read before you start. Edge cases and thinking questions live *inside* the story — *"Consider what happens if two vendors register with the same email…"*. If you want a private checklist for yourself, use Appendix B.

### The blocks a course is made of (code, diagrams, callouts)

A course should *look* like a course, not a wall of prose. Use the full Markdown kit — code blocks, diagrams, callouts, tables — to make each chapter scannable and professional, and **use them generously** (Rule 22): reach for an illustrative block *every time* it would make an idea concrete, rather than describing in three sentences of prose what one annotated snippet would show at a glance. The one rule that never bends: **these blocks illustrate the problem, the contract, or the measurement — they never hand over the solution the mentee is there to write.**

**What earns a code block:**

- **The data shape, not the code that builds it.** An ERD, a table sketch, or a sample JSON payload that shows *what* the data looks like — so the mentee can see the bloated login row or the per-vendor order, then build the schema themselves.
- **The API contract.** The endpoint, method, and request/response shape the feature must honour — `POST /auth/login → { accessToken }`. You're specifying the interface, not the handler behind it.
- **A command to run.** `createdb marketplace`, a migration command, an `ssh` into the box — operational steps, not application logic.
- **Config keys, never secrets.** `.env` *names* (`DATABASE_URL=`, `JWT_SECRET=`) so they know what to set; the values stay theirs and stay out of Git.
- **Tooling configuration the mentee would otherwise copy from docs.** A `tsconfig.json` with the key options annotated, a `package.json` scripts block, a `.gitignore`, a Compose service spec. **Setup and tooling chapters are legitimately code-block-rich** — the "no solution code" rule targets the *feature's logic* (the route handler, the cart-splitting function, the auth middleware), not standard boilerplate. Showing the config and the commands, well annotated, teaches far more than withholding them; just don't cross into writing the feature's actual implementation.
- **The measurement that exposes the cost.** Paste the query log that reveals the N+1, the response that's 2 MB too big, the `git log` line with a secret in it — as the evidence that makes the weak approach's cost *concrete* while you explain it. Show the symptom to prove the point; the mentee doesn't have to build the broken version to see it.
- **Concept pseudo-code on a *different* problem.** If an idea needs a sketch, write it as pseudo-code or against an unrelated example — never the feature's own answer (this mirrors the hint ladder, §16).

**What never goes in a block:** the route handler, the auth middleware, the hashing call, the cart-splitting function — the implementation that *is* the feature. That's the mentee's job; printing it is how a guide quietly turns back into a tutorial (Rule 2, §20).

**The other blocks that make a course readable:**

- **Diagrams.** An ERD, an architecture sketch, or a sequence of the request flow. A fenced ` ```mermaid ` block renders on most course platforms and stays in version control; an embedded image works too. Reach for one whenever a relationship is easier seen than read.
- **Callouts.** Short admonitions for the things that interrupt prose badly: a **Note**, a **Warning** ("never commit this file"), or a foldable **Hint** the mentee opens only when stuck. `> **Note:** …` works everywhere. Keep them rare — a page of callouts is just label spam (Rule 15) in a new costume.
- **Tables.** For anything genuinely tabular: the course outline, a comparison of two approaches, a prerequisites list. Not for prose you couldn't be bothered to write in sentences.
- **The Definition of Done.** Always a checkbox list — the one mandatory block that closes every feature.

The test is the same as everywhere else: does the block help the mentee *decide and build*, or does it decide for them? A contract, a diagram, a measured symptom — those set up the thinking. A finished function ends it.

### The target voice (a feature that climbs — auth API)

This one is worth reading slowly, because it shows both the *ladder* from §1 and the explain-don't-trap voice: at each rung it names the obvious approach, explains *why* it's wrong, teaches the right one in depth, and has the mentee build only that. register → where the details live → split the profile → hash the password → understand auth and choose → add refresh tokens → secure them → verify the account. Each rung is small, and each one earns the next.

It's shown below as one continuous story so you can see the voice in full. In a real course (§7), this is the **Authentication chapter**, and you would **split this ladder across its 20–30 sub-chapters** — *learn it* sub-chapters for "why hashing" and "sessions vs tokens", then *build it* sub-chapters that scaffold the auth folder, hash passwords, add register, add login, issue a token, add refresh tokens, verify the account, each a rung of this same ladder. The voice stays identical; what changes is that the build is delivered as many small, guided steps rather than a few long pages.

> **Feature: Registration & login (API)**
>
> Your database has users and roles, but the server still treats every request as anonymous. This feature is where the marketplace learns who's knocking — and it's the clearest example of a feature that *climbs*: each rung you finish reveals the next.
>
> **Start at the bottom: register.** A user has to become an account, so the first question is the smallest one: what do you store? Create the bare user record — just enough to register and log in. Resist adding more; the next rung will show you exactly where the rest belongs.
>
> **Where does the profile go?** Soon the product wants a display name, an avatar, a shipping address, a bio. The obvious move is to pile these onto the user row — and it's the wrong one. Here's why, concretely: every login reads the user's row to check the password, and now that row drags an avatar URL, a bio, and an address it never needs — on every auth query, forever. The cost is invisible at small scale and real at large scale. The professional shape separates **identity** (what you log in *with* — email, password, verification state) from **profile** (who you *are* — the display fields), because the two are read at different times for different reasons. So you build a lean identity table and a linked profile table from the start, and the mentee never has to construct the bloated version to believe in the split. Have them say, in their own words, why it earns its keep — that sentence is a viva answer.
>
> **Now the password — and the one rule you never break.** The tempting approach is to store the password as it arrives, in plain text. Don't — and here's exactly why: if that table ever leaks (a stray backup, an injection, a curious employee), every account is compromised at once, along with every other site where the user reused that password. Instead you store a **hash**: a one-way fingerprint you can verify a guess against but can't reverse into the original. You add a **salt** — random data mixed in per password — so two identical passwords don't yield the same fingerprint and precomputed attack tables are useless. And you use a hash *built for passwords* (bcrypt, scrypt, or Argon2), which is deliberately **slow** — slowness is the feature, because it makes guessing billions of passwords expensive while costing you milliseconds. *(Worth reading: LinkedIn's 2012 breach leaked 6.5M passwords as unsalted SHA-1 — a fast hash, no salt — and most were cracked within days. It's the clearest argument for "salt, and use a slow hash." Go read what happened.)* The mentee builds it hashed from the first commit; there is never a reason to write the plain-text version.
>
> **How does the server remember you between requests?** HTTP forgets the client the instant the response leaves — every request arrives a stranger. There are two real answers, and the mentee should understand both before choosing. **Stateful sessions:** the server keeps a session record and hands back an id pointing to it — the truth lives on the server, so logging out is trivial, but every server in a fleet must reach that session store. **Stateless tokens:** the server hands back a signed token that *carries* the claims (who you are, your role) and verifies its signature on each request with no lookup — any server can verify any request, but "log out everywhere, now" becomes genuinely hard. *(This is the classic stateless-vs-stateful trade-off — point them at a short "sessions vs JWT" piece and let them read it right here, because it only clicks once you've met both sides.)* Then teach what a **JWT** actually is — a signed, self-describing token of `header.payload.signature`, where the signature (made with your secret `JWT_SECRET`) is what makes tampering detectable — and have them issue one carrying the user id and role. They build the right thing *and* can explain why it's right.
>
> **The gap that appears next: refresh tokens.** A short-lived access token is safer — if it leaks, it expires fast. But make it expire every fifteen minutes and the user is logged out mid-checkout, constantly. The fix is a second, longer-lived credential — a **refresh token** — whose only job is to mint fresh access tokens when the old one dies. And notice it's the password problem returning: a refresh token sitting *readable* in your database is a master key, so it's stored **hashed** too, and logging out must actually **revoke** it — afterwards it stops minting. Explain it, have them add it, and have them prove revocation by trying to refresh with a revoked token and confirming a 401.
>
> **One more rung: verify the email.** A vendor shouldn't open a store with an email nobody owns. So registration doesn't fully activate the account: you issue a one-time code (an **OTP**), deliver it, and refuse to mark the account verified until it's confirmed — and protected actions require a verified account. (If this chapter is growing large, splitting verification into its own short chapter is a fine call.)
>
> Notice the shape — at every rung you named the wrong approach, explained *why* it's wrong, taught the right one, and the mentee built only what's worth keeping. No broken version was ever shipped, and at the top they can explain every decision — which is exactly what the viva asks.
>
> **Definition of Done**
> - [ ] Register and login work for both vendor and customer
> - [ ] Identity and profile live in separate, linked tables — and the mentee can explain why
> - [ ] No readable passwords anywhere in the database; a wrong password is rejected
> - [ ] A protected route rejects a missing, invalid, or expired access token
> - [ ] A refresh token issues new access tokens, is stored hashed, and can be revoked
> - [ ] An account stays inactive until its OTP is verified

That's the target: **a story that climbs, every decision explained in depth (the weak approach *and* the right one, with the why), reading woven through, and a Definition of Done as ticks** — and no *solution* code anywhere in it. The mentee built only the right version of each rung; they never shipped a broken one to learn the lesson. (A contract, an ERD, or a measured symptom shown as evidence would be welcome here; the auth *implementation* would not — see "The blocks a course is made of" above.)

## 10. Prerequisites and readings inside chapters

Reading is not a single block parked at the start of a chapter — it's **woven through the prose** wherever a concept arrives (Rule 18). The pattern is always the same: explain the idea yourself, in plain words, *then* point onward for depth.

- **Start of the guide:** the baseline table (HTTP, Git, SQL, framework basics) — the assumed floor.
- **Inside a chapter, freely:** every time you introduce a concept, teach it inline and add a "worth reading" pointer beside it — sessions vs JWT as you reach auth, hot reads and **stale cache** as you reach the catalogue cache, cursor vs offset as you reach pagination. These can be frequent; a chapter rich with "here's the idea, and here's where to go deeper" reads like a good book, not a thin spec. What you must *not* do is gate the teaching behind the link — explain first, link second.
- **Between chapters:** an optional breather (three to five paragraphs) linking a topic to what they just built — not a separate "module."

Topics that almost always deserve an inline explainer *when your project uses them*: DNS, HTTPS/SSL, stateless vs stateful APIs, authentication vs authorization, N+1 queries, object storage vs database blobs, pagination, your stack's async model, and event-driven patterns if they're relevant. The one rule: every reading must be relevant to the project or the phase the mentee is in. Relevant readings are interesting; irrelevant ones are noise.

A reading is also a natural home for the two *light* touches from §15 — a build-in-public nudge and the occasional real-world insight. Unlike the inline explainers (which can be frequent), keep those two occasional; don't bolt them onto every chapter.

### Reading blocks you can drop in — and hints

Besides the inline pointers above, you have three clearly-headed reading blocks, plus hints, for the moments that call for them — usually *between* chapters, or *inside* a chapter where new knowledge is needed. Use a real, descriptive heading each time, not a generic label. The goal of all of them is the same: **make the mentee read and think for themselves**, so they build their own understanding instead of being spoon-fed.

**1. "Mandatory reads"** — the articles the mentee *must* read at this point, because the next chapter assumes them. Place these **between chapters**, tied to what's coming, so the mentee builds their own mental model *before* they build the code. Keep the list short (one to three strong sources), and give each a one-line reason it's required now. This is the team's headline ask: the mentee learns by reading and thinking, not just by being told.

> **Mandatory reads — before you build auth**
> - **Password hashing (bcrypt / Argon2)** — why you never store or "encrypt" a password, and what a salt solves. *Required: the next chapter builds directly on this.*
> - **JWT, explained** — what a signed token is, and what it can and cannot protect. *Required: you'll issue one in the next chapter.*

**2. "Topics to read"** — *recommended* (not required) deeper dives for the curious, each with a line on why it's worth it. Reach for this when a chapter has outside knowledge that helps but isn't strictly blocking — for example, in an LMS when you reach paid video lessons:

> **Topics to read — protecting your video lessons**
> - **Signed / expiring URLs** — a video link that stops working after a short time, so it can't be shared or reused. *Why: you're about to serve paid lessons and need to stop link-sharing.*
> - **Token-based streaming & DRM basics** — how platforms make paid video hard to simply download. *Why: a plain `<video src>` can be downloaded by anyone who opens browser dev tools.*
> - **Access control on media** — checking the viewer is actually enrolled before the stream starts. *Why: only paying students should ever reach the file — the data-protection part.*

**3. "Interesting to read"** — a true, real-world story tied to what they just built, purely to read and wonder at. It is **never** something they implement; point to a search or a real article, never an invented link (Rule 16):

> **Interesting to read.** Netflix encodes every title into dozens of versions for different devices and connection speeds, and serves them from servers placed *inside* internet providers to cut buffering — search "Netflix Open Connect" to see how.

**Hints — between steps, when something is genuinely tricky.** A hint is a short nudge that helps the mentee *think*, never the answer. It reframes the question, names the concept to reach for, or says what to check — and stops there. Make it collapsible where your platform allows (a foldable "Hint" the mentee opens only when stuck), so it doesn't rob the ones who don't need it.

> **Hint.** Your protected route still lets everyone through? Check *where* you verify the token — is the check running before the handler, or not running at all? (Think about middleware order.)

Rules for all of these: every read must be **relevant** to the chapter, every "interesting" story must be **true** and verifiable (Rule 16), and a hint must **never** contain solution code — it points at the thinking, not the answer. (Interactive checks like quizzes are out of scope for now — focus on these reading blocks and hints.)

## 11. Definition of Done (or Key takeaways)

**Build and setup chapters** end in observable checkboxes — things you or the mentee can **see or demo**:

- Bad: "Implement auth properly."
- Good: "A vendor cannot edit another vendor's product id through the API; it returns 403 or 404."

**Decision and concept chapters** (the ones with nothing to build, like "Why MongoDB?") have nothing to tick, so they end in **Key takeaways** instead — a few "you can now explain…" bullets:

- Good: "You can explain why a document database fits this project, and one case where a relational database would have been the better call."

Every chapter ends in one or the other — never both, never neither.

### The checklist is a gate

The Definition of Done is not a footnote — it's the **gate to the next chapter**. The rule, stated to the mentee in the intro (§8) and worth a one-line reminder at the checklist itself: **don't move on until every box is ticked.** Each item must be something the mentee can genuinely *see or demo*, so "done" can't be faked — that's what makes the gate real. This is what keeps a self-paced course honest: progress is earned chapter by chapter, not skimmed. Phrase the checklist as a real gate (e.g. a closing line: *"All boxes ticked? Then continue to the next chapter — if not, that's where today's work is."*), not as an optional summary.

### Concept-chapter questions: make them probing, and have them written down

For a **decision/concept chapter**, the Key takeaways are the gate — so don't let them be lazy recall ("explain what a JWT is"). Pitch the questions to the difficulty (§3) and make them *demand reasoning*:

- **Scenario walk-throughs** — *"trace a checkout with no transaction; where exactly does the data corrupt?"*
- **Steelman, then counter** — *"give the strongest case for the other approach, then refute it."*
- **Always "use this project"** — make them answer with the actual domain, not in the abstract.

**Structure them in two layers: the decision, then the topics.** First ask one or two questions about the *decision* the chapter makes (*"why this choice for this project?"*); then ask **one question per concept the chapter actually taught** (for "Why relational?" that's transactions/ACID, foreign keys, joins, JSONB — one each). Keep every question **inside the chapter's scope**: test what *this* chapter taught, never something a later chapter introduces. The topic questions are how you confirm each idea the decision rests on actually landed — not just the headline.

A medium–advanced mentee should have to *think*, not parrot the chapter back.

And make the answers **tangible and reviewable**: have the mentee **write them in their repo**, in a **`learning-log/`** folder, one file per chapter (`learning-log/02-why-relational.md`). State plainly that it is **mandatory** — the written answers are the gate's evidence and the mentee's viva rehearsal, and they're what a mentor or AI reviewer reads. Tell the mentee *where* to put them and to commit them. For a **build chapter** the Definition of Done stays demoable, but any "explain why…" item should also be captured in the same log.

## 12. Tone

| Do | Don't |
|----|-------|
| Professional, clear paragraphs | Bullet walls and label spam |
| Headings that name real work | Internal pedagogy labels as headings |
| Explain the weak and the right approach, then build the right one | Make them build the broken version "to feel the pain" |
| Deep chapters that teach the concept | Thin specs that just name the chapter |
| Jargon explained once, in context | Unexplained acronyms |
| A course outline (chapter list) for navigation | A vague outline with no real chapters |
| Code blocks for contracts, data shapes, and measured symptoms | Code blocks that hand over the solution |
| Diagrams and callouts where they clarify | A callout on every paragraph |
| **Intuitive, product-shaped names** — "Create product", "Admin dashboard", "the vendor's shop" | Dry technical labels — "product POST handler", "dashboard page component" |
| **Open with motivation or a story; keep the human side present** | A cold spec the reader has no reason to care about |
| **Concrete build steps — this file, this table, this library, then verify** | "Now implement it" with no folder, file, or guidance |
| **The build broken into many tiny steps**, one action each | One giant "build the whole feature" sub-chapter |
| **✅ do / ❌ don't at each build step** | Only the happy path; the traps left unsaid |
| **Continuity — each step recaps where the project stands** | Steps that read as if the project started from scratch |
| **A chapter is a feature area of 20–30 guided sub-chapters** | A chapter that's one tiny topic, or an undivided wall of prose |

## 13. Closing — ship, document, prove it

The final chapter file (e.g. `99-closing.md`) ends the course with:

- **Deploy** — a real URL, HTTPS, secrets in env (bring in VPS, SSH, DNS, SSL as the story needs them)
- **Document** — ERD, architecture diagram, user flows, challenges and how they were solved, a demo video
- **Case study** — a portfolio or LinkedIn write-up
- **Bar-raiser** — one line: the mentee picks one creative feature or stack swap that doesn't break a core rule (data isolation, for instance)
- **Evaluation** — one line for the mentee: *"You'll walk through how it works and what would break it."*

The full viva structure stays mentor-only (Part V).

---

# Part IV — Teaching mechanics

## 14. Teaching by contrast (explain the wrong, build the right)

Pick the **most common mistake** for that feature and *explain* it — what the beginner reaches for, and why it disappoints — making the cost concrete with a number you describe rather than a broken build you ask for:

- Query count (the N+1 the obvious feed query would fire)
- Response size (the bloated catalogue payload it would return)
- Rows touched (the denormalised vendor name it would copy onto every product)
- Wrong data on screen after a change (the price that would shift on old orders)
- Secrets that would sit in `git log`

State the cost, then teach the right approach and a better alternative in prose, and use **callbacks** so later features lean on earlier ones: *"Remember why one query per product was the wrong shape for the catalogue? The cart faces the same temptation on the frontend — here's how to avoid it."* The mentee builds the right version each time; the wrong one only ever appears as something you explained.

The canonical marketplace example, taught this way: storing **only `product_id`** on an order line *looks* fine — until the vendor changes the price and every old order silently shows the new one, because the line never recorded what was actually paid. So you snapshot the price onto the order line at purchase time from the start. The mentee understands the failure without having to ship an order system that rewrites its own history.

## 15. Scaffolding, pacing, and the threads that run through the guide

### The complexity ladder — start at zero, climb as far as the difficulty allows

A good course goes **from zero to the project's ceiling**, one rung at a time (Rule 20). It doesn't open with the hard stuff — it earns it. The early chapters get the project standing and safe; each later rung adds one more layer of real-world concern. A typical climb:

0. **Stand the project on disk** — repo, `server/` + `client/` folder trees, both apps boot, first health check from browser to API (§7.5). *Nothing else until this is done.*
1. **Get it running** — project setup, configuration from the environment, a health check.
2. **Make input safe** — request validation, sane error handling and status codes. (TypeScript here too, if the stack uses it.)
3. **Model and build the core** — the data model, then the core features as clean, correct CRUD.
4. **Make it correct under real use** — auth and authorization, ownership / data isolation, pagination.
5. **Make it fast and robust** — indexing, caching and invalidation, N+1 fixes, rate limiting.
6. **Make it scale** — async / background processing, batch processing, normalisation vs denormalisation, referencing vs embedding, idempotency.
7. **Ship it** — deploy, HTTPS, secrets in env, basic monitoring.

**How far up you climb is the difficulty you set in §3:**

| Difficulty | Roughly where the course tops out |
|---|---|
| **Easy** | Setup, env config, validation, clean CRUD, basic auth — a correct, running app |
| **Medium** | The above + full auth/refresh, authorization & data isolation, pagination, error handling, basic caching |
| **Advanced** | The above + rate limiting, async/background & batch processing, caching with invalidation, normalisation vs denormalisation, referencing vs embedding, indexing strategy, idempotency |

Two rules for the ladder: **never skip a rung the project genuinely needs** (an advanced course still starts at setup), and **never bolt on a rung the project doesn't need** just to look advanced — every rung must earn its place in *this* app. The ladder is a menu scaled to difficulty, not a checklist to exhaust.

### How much story per chapter

| Stage | How much story |
|-------|----------------|
| First chapters | Full depth — approaches compared, concept taught, reading woven, Done/Takeaways |
| Middle | Shorter context; the mentee knows the pattern, so explain only what's new |
| Last chapters | Leaner — reuse established patterns, but still fully explain anything new |
| Bar-raiser | One line only |

Even the leaner late chapters stay **genuinely guided** (Rule 19): what shrinks is *re-explaining concepts already taught*, not the build steps themselves. "Brief" means you stop re-teaching pagination for the fifth time — not that the build collapses into one coarse "now do it" step. The single genuine exception is the **bar-raiser** (one line by design).

**Pacing:** after a heavy chapter, give a lighter one or a short reading block.

### Pace *within* a chapter, too — easy first, then advanced

A chapter should climb the way the course does (Rule 20): open at the simplest framing and add depth in layers, so the mentee never meets the advanced detail before the basics. For a **tool/setup chapter** that order is:

1. **Frame the options first**, then narrow — *"you could run this as a managed cloud service, install it natively, or use Docker; here's why we pick one for local dev."* Don't drop the mentee straight into the chosen tool as if there were no alternatives.
2. **Introduce the tool from scratch** — what it *is* and its core concepts, with plain analogies, *before* any configuration.
3. **Point to the official install guide** for anything they may not have installed yet (Docker, a CLI, an SDK). Never assume the tool is already on their machine — link the real install docs as a short "get it first" step.
4. **Get the simplest version working**, verify it, *then* layer on the production refinements (version pinning, volumes, health checks, parity).

Leading with volumes and dev/prod parity before *"what is a container?"* loses people. And **mind chapter order** for the same reason: put the thing that stands alone first (e.g. the app skeleton) before the thing that supports it (the services it talks to), unless one truly blocks the other.

### Timeboxing into weeks

When a course has a fixed duration — "4 weeks at ~60 hrs/week" — **group the chapters into weeks** (or sprints), and show those week-blocks in the course outline (§8). Weeks give the mentee a sense of pace and a place to breathe, and they map cleanly onto the complexity ladder above: early weeks at the bottom rungs, later weeks at the top. A rough rule: size each week so it *finishes a coherent stage* (e.g. "Week 2 — Identity, access & isolation"), not an arbitrary page count. State the weekly hours budget once, in the intro, and let the week groupings carry the rest — don't stamp "Day 3" timings onto individual chapters (Rule 14: scope by work, not clock).

A guide is more than a stack of features — it's a journey the mentee lives in for weeks. Four light threads keep them engaged and turn the work into a habit and a story they can tell. The word that governs all four is **lightly**: a sentence here, half a paragraph there, woven into the prose. The moment any of them becomes a heading on every feature, you've rebuilt the form you were trying to escape (Rule 15).

**1. A daily working rhythm, re-surfaced — not stated once.** Set the habits early (commit every day, keep secrets out of Git, read the error and the docs before asking, ask with "here's what I expected vs what happened"), then *remind* them in passing as the weeks go on. A line at the top of a heavier feature — *"Before you start: a clean commit of yesterday's work, and a glance that nothing secret slipped past `.gitignore`."* — does far more than one rules box nobody rereads.

**2. Small reminders and callbacks between features.** A sentence or two that links back to what they just built keeps the thread tight: *"The catalogue taught you to batch your queries — hold onto that, because the cart will tempt you into the very same N+1 on the frontend."* These double as pacing: a breather and a memory jog between two heavier features.

**3. Real-world insights — interesting, not assigned.** Every few features, drop one true story about how a real team solved the problem in front of them, purely to read and wonder at: *"Stripe makes payment requests safe to retry with idempotency keys, so a flaky network never double-charges a card — worth a search before you design checkout."* or *"Netflix deliberately kills its own servers in production with a tool called Chaos Monkey, to prove the system survives — read about it when you deploy."* Three rules: it must be **true** and verifiable (never invent a company's design), it is explicitly **not** something they implement, and you point to a search or a real article rather than a made-up link.

**4. Build in public — branding what they learn.** Right where a reading sits, nudge the mentee to turn what they just understood into something with their name on it — a short LinkedIn post, a piece on their portfolio site, a dev-blog article, or a thread, whichever they prefer — using their own feature as the worked example. None of it is mandatory, but by the end they've shipped an app *and* a trail of write-ups that read like an engineer thinking out loud, which is the portfolio that actually gets them hired. Keep it to the moments that genuinely earned a write-up, not every feature.

None of these four is mandatory on a given feature; they are seasoning. Use them where they land and skip them where they'd clutter.

**Weave in the technical themes too, where relevant:** Git discipline, security and data isolation, performance, accessibility, privacy, "normalise then denormalise with trade-offs", and deployment secrets.

## 16. When mentees are stuck *(mostly mentor; one line in the intro)*

The mentee guide intro can say: read the error, check the docs, explain the problem out loud, try a smaller test, take a break, then ask with what you expected vs what happened.

**Your hint ladder (never give solution code):** reframe the question → name the concept to read → describe the strategy in words → show a tiny example on a *different* problem → escalate.

The *ladder* — how you escalate help in person — stays yours, mentor-only. But individual **hints** are welcome in the mentee guide itself: a single nudge that reframes the question or names the concept to reach for, dropped (ideally collapsible) where a step is genuinely tricky. See §10 for how to write one. A hint helps the mentee think; it never shows the answer.

---

# Part V — Assessment

## 17. Viva, documentation, tiers

**Documentation** — use cases, ERD, architecture, user flows, challenges, a demo video.

**Viva** — the mentee hears: *walk through how it works and what would break it.* You use MCQs, open questions with follow-ups, and scenarios. Score each competency 0–3 (mentor only): 0 can't explain · 1 the *what* · 2 the *why* · 3 the *trade-offs*. Pass: mostly 2s, with some 3s on the core competencies.

**Tiers for the mentee:** minimum pass (core features + can explain them) · target (full scope + docs) · stretch (the bar-raiser).

## 18. AI tools and understanding

Mentees may use Cursor, Copilot, and the rest — the guide doesn't ban AI. Evaluation checks that they understand what they shipped: in the viva they explain auth, isolation, queries, and trade-offs. If they can't explain code they committed, *that* is the signal — not whether AI was involved. If your program wants it, a private AI journal is fine; it doesn't need to clutter the mentee-facing guide.

---

# Part VI — Quality control

## 19. Pre-publish checklist

**Shape**

- [ ] Course is organised as **phases → chapters → sub-chapters** (§7); the intro is the `01-introduction/` chapter (not a `README.md`)
- [ ] A **chapter is a feature area** (~12–18 total), broken into **20–30 sub-chapters** following the arc (set the scene → learn it → reading → build it in basic steps → checklist), not one undivided wall (Rules 4, 19, 24)
- [ ] The **Build it** cluster is **many tiny steps** (scaffold → write → wire → test), each with **✅do/❌don't** and a continuity recap (Rule 27)
- [ ] Chapters, sub-chapters, and features are named **intuitively and product-shaped** ("Create product", "Admin dashboard"), never dry/technical or vague ("Module 4", "dashboard component") (Rules 5, 26)
- [ ] **Project skeleton chapter** exists early (§7.5): `server/` + `client/` trees, both apps run, client hits health API
- [ ] **Backend and frontend chapters paired** per feature where both exist; each phase ends with something **demoable** (§7.6)
- [ ] **No run of 3+ concept-only chapters** without a runnable build step between them (Rule 28)
- [ ] Intro chapter order: Introduction → **How to use this course (the checklist gate)** → The project, then actors → In/Out of scope → Prerequisites → Course outline → Working rhythm
- [ ] Headings sit on their own line (not bold lead-ins buried in paragraphs)
- [ ] Introduction short; no fear spam
- [ ] Course outline lists chapters in order, grouped by week/phase, each linked to its file
- [ ] Prerequisites are relevant, necessary, and **easier than the course** — nothing the course itself teaches; pitched to the difficulty (§8)
- [ ] The checklist-gate rule is stated in the intro, and each chapter's Definition of Done reads as a gate, not an optional summary

**Each chapter**

- [ ] Written as a **story** — the weak approach and the right one explained with the *why*; the mentee builds only the right one
- [ ] **Deep enough to teach the concept** (Rule 17), and genuinely **fine-grained** — every build step its own sub-chapter (Rules 19, 27) — not a thin spec or a coarse "now build it"
- [ ] Has a real **Build it** sub-chapter that **scaffolds the project step by step** — names the exact folders/files, the table/column, the library to choose, the order of steps, and how to verify — **without** handing over the solution code (Rule 25, §9.6). Concept chapters connect via applied work on the real codebase (Rule 23)
- [ ] Opens with a **human, intuitive hook** (motivation, a story, why it matters), not a cold spec (Rule 26)
- [ ] Where the chapter offers *legitimate* alternatives, it gives **pros/cons and a clear, mandated choice** — foundational decisions aren't left open (§9)
- [ ] The mentee is never told to build the broken version "to feel the pain"
- [ ] No repeating template labels (Guardrails, Reflection, etc.)
- [ ] Jargon explained in prose the first time; concepts taught inline, links to deepen (Rule 18)
- [ ] Ends in a **Definition of Done** (build/setup chapters) or **Key takeaways** (decision/concept chapters)
- [ ] **Generously illustrated** with the necessary blocks (data shapes, contracts, DDL/config sketches, commands, comparison snippets, a diagram) — not prose-only (Rule 22)
- [ ] Code/diagram/callout blocks illustrate the spec, contract, or measured cost — never the solution
- [ ] No feature solution code (Rule 2)
- [ ] Reading woven through the prose, relevant to the chapter — explained first, linked second

**Whole guide**

- [ ] Each chapter teaches by contrast (weak vs right, explained), with at least one callback to an earlier decision
- [ ] Chapters **climb** — each step reveals the next problem rather than listing them all at once
- [ ] **Long and detailed** — ~12–18 feature-chapters across phases, each broken into 20–30 guided sub-chapters (Rule 19); no feature crammed into too few steps; nothing reads thin **or** coarse
- [ ] **Starts at zero and climbs the complexity ladder** to the level the difficulty sets — setup → env → validation → core → the advanced rungs (§15); no needed rung skipped, no needless rung bolted on
- [ ] **Mandatory reads** placed between chapters where the next chapter depends on them
- [ ] **Hints** appear where a step is genuinely tricky — a nudge, never the answer
- [ ] The daily working rhythm is re-surfaced through the guide, not stated once and forgotten
- [ ] Real-world insights and build-in-public nudges appear lightly — never as a repeating heading
- [ ] Scaffolding fades; the bar-raiser is minimal
- [ ] No mentor-meta on the mentee's page
- [ ] Closing covers deploy, docs, bar-raiser, and the evaluation line

## 20. Smell tests

**Mentor-meta leaked?** Delete the personalisation menu, competency map, rubric, "this module teaches…".

**Tutorial by accident?** Solution code · nothing for the mentee to reason about · a keystroke walkthrough.

**Make-them-fail by accident?** The chapter tells the mentee to build the broken version "to feel the pain" instead of explaining the weak approach and having them build the right one.

**Too thin?** A chapter that just names the topic and lists a few bullets · concepts named but never explained · teaching gated behind "go read this" with nothing taught in the chapter itself (Rules 17–18).

**Too short, or wrong altitude?** The whole course is ten thin pages (Rule 19) · an *advanced* project never climbs past basic CRUD · an *easy* project is buried under rate limiting and sharding it doesn't need · no mandatory reads where the next chapter clearly depends on prior knowledge (Rules 19–21).

**Form guide?** Every chapter carries the same ten subheadings · it reads like a worksheet, not a real course.

**Wrong structure?** A vague outline · no course outline / chapter list · prerequisites missing · a chapter written as one undivided wall instead of **20–30 sub-chapters** (Rule 24) · a "chapter" that's one tiny topic instead of a **feature area** (Rule 4) · no phase/mindset framing. (Layer names like "Backend" are fine — *empty* chapters are not.)

**Teaches but doesn't build? (the big one)** The chapter explains the topic well but never **guides the mentee to do it on the project** — no folder/file named, no "add the index to *this* table", no "choose *this* state library", no order of steps, no way to verify. The connection from *learning* to *a working project* is missing (Rules 25, 27, §9.6). This is the most common and most damaging failure: good content that doesn't ship a project.

**Concept course, empty product? (Rule 28)** The guide teaches normalization, RBAC, threat models, and caching in depth, but the mentee's repo has no client, no checkout, or no demoable feature for half the course · three or more chapters in a row add only docs/learning-log work · concept chapters appear *before* the feature they describe instead of after or adjacent to implementation (§7.6). The mentee could pass a viva on definitions but cannot **run the app** — wrong balance.

**Build too coarse or disconnected?** The hands-on is one giant "now build the feature" step instead of **many tiny sub-chapters** (Rule 27) · no **✅ do / ❌ don't** at the steps, only the happy path · steps that read as if the project started fresh instead of **continuing from its current state** (Rules 23, 27). Basic, sequenced, connected — or it doesn't get built.

**Dry and lifeless?** Cold, all-business chapters with no human side · technical names ("the product POST handler") instead of intuitive ones ("Create product") · no motivation or story to make the mentee *want* to build it (Rule 26).

**Bad readings?** Invented URLs · a reading block on every chapter when nothing new is needed.

**Everything dumped at once?** A wall of requirements at the top of a chapter instead of one problem unfolding into the next — the chapter should climb, not list.

**Engagement turned to clutter?** Reminders, insights, or build-in-public prompts have hardened into a repeating heading or boilerplate on every chapter, instead of landing lightly where they're earned.

## 21. Reviewers

Three or four readers, senior and junior. The senior finds the missing pieces; the junior finds the vague, unexplained steps. **Build it from the guide yourself** before you ship.

---

# Appendix A — Glossary (plain English)

Use these inline in your guides; this appendix is your backstop.

| Term | In plain words |
|------|----------------|
| **HTTP / HTTPS** | Rules for browser–server talk; HTTPS adds encryption in transit. |
| **REST / API / Endpoint** | A standard style for web APIs; one URL you call. |
| **API contract** | The agreed shape of a request and its response for an endpoint. |
| **ERD** | Entity-relationship diagram: a picture of your tables and how they link. |
| **DNS** | Internet name → address lookup for your domain. |
| **JWT** | A signed token proving who you are on each request. |
| **Hash / Salt** | A one-way password fingerprint; salt stops identical passwords matching. |
| **Authentication / Authorization** | Who you are vs what you're allowed to do. |
| **Middleware** | Code that runs between the request arriving and the response leaving. |
| **Foreign key / Join** | A link between tables; combine them in one query. |
| **N+1 queries** | One query per list item instead of one batched query. |
| **Normalisation / Denormalisation** | Store each fact once vs duplicate it for speed (a trade-off). |
| **Pagination** | Results in pages, not all at once. |
| **Cache / Cache invalidation** | Fast reuse of answers; refresh them when the data changes (avoid stale reads). |
| **Object storage / CDN** | Files served by URL; copies kept near users for speed. |
| **Multi-tenant** | Many owners on one system; each must see only their own data. |
| **Viva** | A spoken exam: explain and defend the build. |
| **Definition of Done** | Tickable, observable outcomes that close a build/setup chapter. |
| **Key takeaways** | The "you can now explain…" bullets that close a decision/concept chapter. |
| **Teaching by contrast** | Explain the weak approach and why it fails, then the right one and why — and build only the right one. |
| **Project skeleton** | The early chapter that creates `server/` + `client/` folder trees, boots both apps, and connects them — before deep concepts. |
| **Tutorial hell** | Copy-only learning that never teaches the decisions. |

---

# Appendix B — Mentor checklist per chapter *(never print on the mentee's page)*

While drafting, make sure each chapter has:

- [ ] A clear point, in plain language — the decision, the setup, or what the feature must do
- [ ] The weak approach **explained** (not built) with its concrete cost, and the right approach beside it
- [ ] The root-cause "why" and a comparison of the alternatives, taught in depth
- [ ] The concept genuinely taught inline — enough to learn from, not just a name and a link
- [ ] Reading woven in where concepts arrive (explained first, linked second)
- [ ] A behaviour spec + a **Definition of Done** (build/setup chapters), or **Key takeaways** (concept chapters)
- [ ] Concept chapters scheduled **after or adjacent to** the build they explain — not a long runway before the repo moves (Rule 28, §7.6)
- [ ] **Build it cluster** names folders/files, sequences steps, includes ✅do/❌don't, and ends each step with verification (Rules 25, 27)
- [ ] After this chapter, the mentee can **point at something new in the repo** or **demo something running** (Rule 23)
- [ ] Edge cases thought through inside the prose
- [ ] No solution code; the mentee builds only the right version

---

*Ship guides where the mentee thinks, compares, reads, builds the right thing, and can explain it — taught in depth one rung at a time, the weak approach and the right one laid side by side, **with a repo that keeps moving toward a finished app** — folder by folder, file by file, screen by screen — carrying the habits, the curiosity, and the mentee's own voice all the way up, not dumped as an internal form.*

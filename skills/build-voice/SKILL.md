---
name: build-voice
description: Build a voice profile from dictated samples, one register at a time — the cut-don't-compose method. Use when the user says "build a voice profile", "run the voice build", "build [name]'s voice", "run a register run", "cut this dictation", "voice engagement for a client", or asks to teach Claude to write like a specific person. Also use when an existing voice file produces drafts that do not sound like their subject. Screens sources for AI contamination first.
---

# Build voice

Produce a voice profile that makes a person sound like themselves. Not like a competent stranger doing an impression of them.

The method is the opposite of the obvious one. **You do not derive a voice from someone's published archive.** You get them talking, you cut the transcript down, and you never write a sentence. Everything below exists because the obvious method was tried first and failed for a reason that is not obvious.

Full narrative, evidence, and the numbers behind every rule here: `Voice-Build_Repeatable-Process_v1.0` in the voice-build workspace. This skill is the operating procedure; that document is the argument.

## ⚠ Do not chain to `voice-builder`

The account-level `voice-builder` skill builds a profile from "3 to 5 sample pieces of writing" and auto-starts on load. That is the contaminated method. It shares most of its trigger phrases with this skill, so **if `voice-builder` fires on a request that belongs here, stop it and say why.** Its outputs are not inputs to this skill.

`newsletter-voice` is different and is a legitimate downstream step — run it after this skill for a subject who publishes a newsletter, per Phase 6.

## Config

This skill reads instance values from `solo-os-config.json`. Locate it by searching the connected Cowork project folder(s) for a file named exactly `solo-os-config.json` and load the first match — do not assume a specific folder name or path.

Used if present: `voice.sources`, `voice.newsletter_voice_path`, `voice.style_guide_path`, `user.name`.

**Config is not required to run this skill on a new subject.** The subject's identity, registers, and file locations are collected in Phase 0 and confirmed with them. Config only tells you where the *config owner's* own stack lives, which matters when the subject is the config owner and not otherwise.

Ask for and confirm: the subject's name, the output folder, and whether this is the config owner or a client. Never write a client's voice files into the config owner's `About Me/` folder.

---

## Phase 0 — Intake and the contamination screen

### Step 0 — Check whether a profile already exists. Stop if it does.

**Before anything else, look for an existing voice profile.** Check `voice.sources` and `voice.style_guide_path` in config, and search the target folder and the connected project folders for `voice.md`, `about-me.md`, and `voice-exemplars.md`.

**If any of them exists, stop and ask.** Do not run the contamination screen, do not collect samples, and do not write a file. Say what you found and where — filenames and paths only, and the version line if the file carries one in its first few lines. **Do not read further into it than that.** Then ask which of these is happening:

- **A new subject** who happens to share a workspace with someone else's profile → confirm a separate output folder before continuing, and never read the existing profile as a template.
- **A new register for an existing subject** → this is not a fresh build. Run Phase 2 only, and compile into the existing file rather than beside it.
- **A rebuild of an existing profile** → confirm explicitly, archive the current file with a dated `-ARCHIVED` name first, and say which version is being replaced.
- **A mistake** → stop.

**Why this gate exists.** Phase 3 writes `about-me.md`, `voice.md`, and `voice-exemplars.md`. Run without this check on a subject who already has a profile, this skill produces a second file with the same name claiming the same authority — which is the exact failure the install phase below exists to prevent, manufactured by the tool that prevents it. **Pointers, not copies, applies to this skill's own output.** It was found on the first live run, on 2026-08-28, by running the skill rather than reviewing it.

**Never resolve this by reading the existing profile and continuing.** A profile in the folder is evidence about a different person, or an earlier version of this one; either way it contaminates what the dictations are supposed to produce independently.

---

**The intake question is never "what have you published." It is "what have you written with no AI involved at all, and can you prove it."**

Explain the reason in one line before asking, because subjects who understand it start flagging their own provenance and the process gets better fast: *approval is not authorship.* People approve and publish drafts containing habits they will reject the moment you show them the habit on its own.

### Grade every candidate source

| Grade | What it is | What you may take from it |
|---|---|---|
| **A — Clean** | Dictated, or written with no assistant, provenance stated by the subject *before* the work | Everything — sentence-level and structural |
| **B — Outline-contaminated** | Every sentence theirs; the skeleton came from an AI-assisted outline or template | **Sentence-level findings only.** Every structural finding is asterisked and cannot become a law on this evidence alone |
| **C — Edited** | They wrote it, an assistant polished it | Subject matter and argument. Nothing about rhythm, endings, or sentence shape |
| **D — Assisted** | Drafted with an assistant, approved by them | Subject matter only. Quarantine it, and say so in the file |

Grade B is the one that gets missed. An outline sets section count, section order, where a piece breaks, and what the organizing unit is, without touching a word.

### The absence test

Contaminated writing is identified by what is missing, not what is there. Once you have two or three confirmed tics, screen every remaining candidate for them. **A sample with none of the confirmed tics is suspect regardless of who typed it.** Assisted prose has no fingerprints — the tics are not diluted in it, they are absent.

Set up a `clean-samples/` folder and a `suspect-samples/` folder. Number every sample. Write the grade and the provenance into the file itself.

**Do not clean any transcript.** Disfluency, repetition, false starts, and odd constructions are the signal. A tidy-up pass converts their prose into model prose before the work starts.

---

## Phase 1 — The register map

Ask what they actually publish, and scope to that. A subject who only writes client email does not need a LinkedIn register.

Then ask the question that saves an entire register run:

> **Which document types do you already write in something other than your own voice, and why?**

Most experienced writers have a second register they have never named. The answer becomes a routing rule and removes those documents from this build's scope. Record it; do not argue with it.

Produce the register map as a table: register / does this file govern it / which file governs it if not. Confirm with the subject before running anything.

---

## Phase 2 — The register run (repeat per register)

**One register per session.** Do not batch them and do not compile until at least two are done.

### Step 1 — Name the question this run answers

Every run should be trying to settle something the last run could not. Write it down before dictating. A run with no question produces a sample and no findings.

### Step 2 — Dictate

Give the angle, nothing else. No structure, no questions, no interruptions. Eight to twenty-five minutes depending on register. **Run it on a real deliverable they actually owe someone** — real work surfaces preferences an exercise never will.

Before they start, ask them to grade the provenance of what they are about to produce. In advance, not after.

Freeze the raw transcript as a numbered sample. **Never edit a frozen sample again** — every later fix lives in the cut draft.

### Step 3 — Cut, do not compose

Cut, trim, resequence. Do not write a sentence.

**The one allowance:** composed bridge sentences, for flow only. **Never a claim, a number, an opinion, or a specific the subject did not say.** Flag every composed sentence inline so they can strike it. Report the count — "composed sentences: 0" is the target and is achievable at 2,400 words.

If you find yourself composing to make a section work, the section is not in the transcript. Cut it instead.

### Step 4 — List every fix. Fix nothing silently.

Two tables in the cut draft:

- **Silent, by prior agreement.** One narrow pre-authorized class only. Agree it with the subject in Phase 0 and never widen it mid-run.
- **Listed, for decision.** Everything that changes a word. Was / Now / class / basis. They rule on each one.

**Check a suspect noun against the system before smoothing the sentence that contains it.** A dictation error that looks like a typo is often a factual error about the thing being described. The obvious fix makes the sentence internally consistent and leaves it describing something that does not exist. This costs two greps and it is the rule that earns the whole listing discipline.

**Leave errors that are the subject.** Agreement and grammar errors that are how the person talks stay in their own writing; fix them only in client deliverables. List what you left and why.

### Step 5 — Write the register findings, in the cut draft

Three buckets, and the third is the one that gets skipped:

- **Carried** — patterns confirmed from an earlier register. State the count: three of three is a different claim from one of one.
- **New to this register** — and say whether it is sentence-level (survives a Grade B asterisk) or structural (does not).
- **Did not hold** — a law that failed here was over-scoped there. This is the most valuable bucket in the run.

### Step 6 — Record the cut rate

Words in, words out, number of cuts, number of composed sentences. One line.

**The cut rate is a property of the register, not of the method.** An argument has run-up and restatement to remove; a process description does not. A 4% cut has not failed. **Never push for a target percentage — you will start composing to hit it.**

---

## Phase 3 — Compile

Only after two or more registers exist.

### The scoping gate — a law is inadmissible without this

**No law enters the file until it has been checked against every clean register on file.** Not named. Checked. It is one pass and it costs less than a version bump.

Every law's entry states three things:

1. **Observed in** — register, sample number.
2. **Checked against** — the other registers by name, with the result for each: fires / absent / fires differently.
3. **Scope** — general, or named registers only.

One sample is an observation. Two sources sharing no inputs is a property of the writer.

**This is a gate because the note version does not work.** On the reference build, two laws shipped with the wrong scope in two days — one too broad, one too narrow — after the rule to name provenance was already in the file. Both named their provenance correctly. **Naming provenance is not the same as checking it.**

### The layer gate — decide ⬡ or ◆ before writing the rule

**⬡ ARCHETYPE** — portable, belongs to the craft or the form. Ships to the next subject.
**◆ INSTANCE** — this person's registers, tics, rates, proof points, refusals. Never ships.

Most laws are split: the mechanism is archetype, the content filling it is instance. "Name the actual tool" is ⬡; the tool list is ◆.

**Test:** would this be true, in this form, for a different person? If yes it is ⬡ even with their example attached. If it only survives because it is them, it is ◆.

Decide before writing, not after. A rule filed in the wrong layer is how the next client ends up sounding like this one.

Seed the ⬡ layer from `references/archetype-layer.md`, which carries the archetype laws already confirmed across three registers. **Confirm each one against this subject's own samples before keeping it** — a seeded law is a hypothesis, not a finding.

### The contradiction sweep

Before calling a version done, read the file against every sibling file governing the same output. **Look for two rules that disagree, not for one rule that is wrong.**

The failure mode of a rules file is never a wrong rule. It is two rules that disagree — nobody reads a 20,000-character instruction file end to end looking for collisions, including the model applying it, which satisfies whichever rule it met most recently and looks compliant doing it.

**Grep the shared surface feature, not the rule name.** Two rules about sentences containing "not." Two rules about hedging. Two rules about the same document type. Two rules about the same punctuation mark. Also sweep the phrase bank against the writing laws — a bank recommending a construction the laws ban forty lines earlier is the commonest instance of this.

### Measure and prune

State the loaded token cost of the whole stack in the file, at every compile. A voice stack is loaded before a word is drafted; it is a standing cost on every draft forever. Set a ceiling. When it is hit, **swap rather than append.** Prune over append.

### Output files

| File | Contents |
|---|---|
| `about-me.md` | Identity, background, working preferences — only what affects voice, taste, metaphors, or judgment. Not a bio. |
| `voice.md` | The operational file. Instruction-shaped, not descriptive. Sections and their layers are listed in the process doc §6. |
| `voice-exemplars.md` | Complete pieces, verified clean, spread across registers. |
| *(optional)* one register-override file per publishing channel with its own rules | Reached by an explicit path, kept out of the general sources list so it cannot bleed into the wrong register. |

**The exemplar rules:**

- **Five. Not three, not twenty.** Fidelity roughly doubles from zero to five and flattens after. A sixth costs context and buys nothing.
- **Spread across registers, never topic-matched.** Selecting by topic similarity measurably *reduces* fidelity by narrowing the stylistic range being copied. This is counterintuitive and it is the mistake most people make.
- **Complete pieces, not excerpts.** The arc is part of what is taught.
- **Provenance at the top of the file.**
- **Swap, never add.**

**A clean exemplar set needs no warnings block.** If the file has to tell the reader which habits not to copy from its own exemplars, the exemplars are contaminated. Replace them; do not annotate them.

### Anonymization

Anonymize **by size and function, never by industry niche.** A niche descriptor next to a named consultant and a dated project is the identification, not the anonymization. This matters more than it looks: a voice file's sample anonymization gets copied verbatim by other systems reading the file, and on the reference build exactly that happened.

---

## Phase 4 — Validate by running it, not by reviewing it

**Reviewing a voice file finds nothing. Running it against real documents finds everything.**

1. **Audit two or three real documents** the subject already wrote or published — ideally written *before* the laws existed. For each law record: fired correctly / did not fire / fired wrongly / no occasion. This is the only step on the reference build that has ever caught a wrong law, and it caught one within four hours of a compile that a careful reading had passed.
2. **Draft something fresh and hand it over saying nothing.** Do not explain the choices. The unprompted reaction is the test.
3. **Log every objection verbatim.** Each is a missing law or a wrong law.
4. **Fix and repeat once.** If the second draft still draws *structural* objections, the sampling was too shallow — run another register. Do not patch the file.
5. **Confirm the anti-overfitting guard is live.** A draft that is a performance of the voice file is as wrong as one that ignores it.

---

## Phase 5 — Install. This step is not optional.

**A voice profile is not a document. It is a dependency.** Getting the artifact right and getting it into the path where work happens are two different jobs, and only one of them feels like progress.

Before calling any version done:

1. **Search every connected project folder by filename** for anything claiming to be a voice profile — `voice.md`, `about-me.md`, `newsletter-voice.md`, and near-misses.
2. **Read every instruction file that routes to one** — `CLAUDE.md`, skill configs, agent prompts, project instructions.
3. **Confirm every path resolves.** An instruction pointing at a folder that is not connected fails silently, produces no error, and never gets fixed.
4. **Replace every copy with a pointer stub** naming what it was, where canonical lives, and why the copy is gone.

**Pointers, not copies.** A copy is how a stale profile survives while the real file moves underneath it. On the reference build, a publishing project ran a five-week-old profile — built from the exact contaminated posts the rebuild existed to escape — while the canonical file moved through four versions. It surfaced because someone asked whether the thing was plugged in, not because any audit went looking.

Report what you found, including "nothing stale found," which is also a result.

---

## Phase 6 — Handoff and maintenance

Deliver the maintenance instructions, not just the files. A voice file that is never updated is fighting last year's version of the person.

Hand over: the scoping gate, the contradiction sweep, Phase 5, and the change-log rule.

**Every change gets logged, including trivial ones.** An unlogged change is invisible, and this whole method exists because of edits nobody tracked. A silent "correction" can put a draft out of compliance with the file's own rules and leave no trace of how.

**Keep a story log alongside the change log.** Different documents. The change log records what changed; the story log records what was tried, what broke, and what the numbers were. Never rewrite an entry to make the path look straighter than it was — the wrong turns are the only evidence about the method.

**Quarterly:** reread the last ten drafts. Add patterns that now read machine-written. Delete rules that no longer trip anything. Swap an exemplar only if a newer clean piece beats one of the five. Re-measure the token cost. Re-run Phase 5 whenever a project moves or a new one starts drafting.

**If the subject publishes a newsletter,** run `newsletter-voice` after this skill to build the channel register on top of the general profile. Keep it out of the general sources list so it cannot bleed into the other registers.

---

## What this skill does not do

State these plainly rather than implying coverage that does not exist.

- **The interview blocks are largely untested.** Three of four planned blocks were harvested by doing the work instead. Run the beliefs-and-contrarian-takes block if the subject has the time; **do not gate delivery on it.** A profile can ship carrying one line on what its subject will defend in an argument.
- **Reacting beats being asked, and working beats both.** Do not run a writing-mechanics interview — it produces a worse answer than draft-reaction and then anchors the subject to it. Show a draft; record objections verbatim.
- **A subject with no clean written samples at all is untested.** Weight the dictations more heavily and make the aesthetic-crimes questions primary. Say up front that this case has not been proven.
- **Everything here comes from one subject.** Every archetype claim is a bet that a finding from one person generalizes. Expect some of them to turn out to be instance in disguise, and log it when they do.

---

## Hard rules

- **Only the subject's own sentences count.** Published AI-assisted work is quarantined, read for subject matter, never for structure or endings.
- **Cut, do not compose.** Bridges for flow only, flagged every time, and never a claim, a number, an opinion, or a specific they did not say.
- **Never edit a frozen sample.** Fixes live in the cut draft.
- **List every fix. Fix nothing silently** outside the one pre-agreed class.
- **No law enters the file until it has been checked against every clean register.** Not named — checked.
- **Decide ⬡ or ◆ before writing the rule down**, never after.
- **Never ship an instance layer to a second subject.** Not with the names swapped, not as a starting template.
- **Never write a profile beside an existing one.** If `voice.md`, `about-me.md`, or `voice-exemplars.md` already exists in the target, stop and ask — see Phase 0 step 0. Two files claiming to be the voice profile is the failure this skill's install phase exists to prevent.
- **Run the contradiction sweep before calling any version done.**
- **Never chain to `voice-builder`.** Its method is the one this skill exists to replace.
- **Log every change, including trivial ones.**
- **The subject's spelling convention beats any invoked skill's**, and gets checked after every structure or humanizer pass.
- Sanitize: no client names or confidential detail in any sample, cut draft, or finding — including in provenance notes.

# Pulse

**A prototype for resolving learner identity across disconnected feedback systems.**

Learner feedback lives in six systems that were never designed to talk to each
other. A support agent looking at an angry ticket has no idea the same learner
rated their last three sessions 2/5. A programme manager reviewing a cohort's
survey scores can't see that a quarter of those learners have open tickets.

Pulse answers one question: **given a learner, what does every system know about
them?** You search by email, ticket ID, or cohort and get one consolidated
profile — including an explicit account of what couldn't be matched.

> **This is a prototype built on synthetic data.** No production data,
> credentials, endpoints, or system names appear anywhere in this repository.
> Source systems are genericised. The data generator in `seed/` is the only data
> source. See [Project status](#project-status) for what runs today.

<!--
SCREENSHOT GOES HERE — replace this comment with:
![Pulse learner profile](docs/images/profile.png)
A single image of the search box resolving a learner, with the unmatched-records
panel visible, does more than every section below it.
-->

---

## Who this is for

Two users, two different moments:

**The support agent, mid-ticket.** They have a learner in front of them and
sixty seconds before they need to reply. They want to know: has this person
complained before, how are they actually doing in the programme, and is this an
isolated irritation or the fourth thing that's gone wrong this month. Today that
means opening four tabs, and mostly they don't bother.

**The programme manager, reviewing a cohort.** They're deciding where to spend
limited intervention capacity. Aggregate satisfaction scores tell them a cohort
is fine; they need to know which eleven learners aren't.

Both are lookup problems, not reporting problems. That shaped the entire
interface.

## Why this is harder than it sounds

There's no shared primary key across the source systems. Email is the only
field appearing in more than one place, and it isn't reliable:

| Source | Learner identifier available | Usable as a join key? |
| --- | --- | --- |
| LMS content ratings | email, program ID | Email yes, with normalisation |
| Live session ratings | email, cohort ID | Yes, with normalisation |
| Support desk A (delivery) | *(none)* | **No.** Blocking gap |
| Support desk B (student) | email, helpdesk user ID | Email yes; user ID unverified |
| Survey platform | email, cohort name | Yes, with normalisation |
| Reporting layer | learner ID, email | Yes |

Note what the second column does and doesn't contain. Several sources carry a
`program_id` or `cohort_id` — plenty of context about *what* a learner is
enrolled in, and no stable identifier for *who they are*. Programme context is
not identity, and a system rich in the former can still be unable to tell you
whether two rows describe the same person.

Two problems dominate the design:

**Support desk A has no learner identifier at all.** Not a weak one — none.
Tickets from it cannot be attached to a learner profile by any means available
in the data. This is unresolved and needs a change at the source system, not a
cleverer query.

**Support desk B exposes a `User Id`** that looks like a learner identifier but
sits in a different namespace from the LMS learner ID. It is loaded and
deliberately not used as a join key.

Beyond that: emails arrive mixed-case, whitespace-padded, sometimes blank;
cohort IDs are missing on some feedback rows; tickets exist for people absent
from the learner master; one export contains genuinely duplicated column names.

## Product decisions

The parts worth arguing about.

### Search-first, not dashboard-first

The first design was a dashboard — cohort health, rating trends, ticket volume.
It was the obvious thing to build and it was wrong. Nobody opens a tool like
this to browse aggregate statistics. They open it because they have one specific
person in front of them and a decision to make in the next minute.

The home screen will be a query box. Aggregates still exist, but one level
down, and they exist to answer "is this learner unusual" rather than to be
browsed.

**Cost of this choice:** it demos badly. A dashboard screenshots well and looks
substantial. A search box looks like nothing until you type into it.

### The first slice is an aggregate view anyway

The first thing built on top of the data is a ratings view for the programme
manager: rating distribution, average rating by cohort, the most-cited reasons
for low scores, the modules that need attention, and the low-rated comments
behind them. That looks like a reversal of the decision above. It isn't.

Learner search is only worth having once at least two sources can be attributed
to the same person. With one source, a "profile" is just a filtered table. A
ratings view is useful with one source, and it forces the whole pipeline —
ingestion, staging, cleaning, rendering — to work end to end on real-shaped data
before anything depends on cross-source joins. Search becomes the home screen
when there is something to consolidate.

**Cost of this choice:** the first thing anyone sees is the kind of screen I
argued against. The mitigation is that it is scoped to one user moment (the
manager's review) and every view ends in a list of specific modules and comments
rather than a headline number.

### Flag, don't impute

When a join fails, a key is missing, or two sources disagree, Pulse says so. It
doesn't fill gaps with best guesses, quietly drop unmatched rows, or pick a
winner between conflicting values.

A profile showing "3 tickets could not be matched to this learner" is more
useful to someone making a decision than a tidy profile that's quietly wrong. In
a support context, confidently wrong is worse than visibly incomplete — an agent
who trusts a clean profile and is wrong twice stops trusting the tool entirely.

The same rule applies to aggregates. Feedback rows with no cohort are kept under
an explicit "(no cohort)" label and counted in a data-quality panel, rather than
disappearing from the totals — which is what a naive pivot does by default.

**Cost of this choice:** the interface is busier and less reassuring. Coverage
numbers look worse than a system that silently drops unmatched rows would
report. That's the honest number, and it's the one that tells you what to fix.

### A minimum sample before anything is ranked

A module with one 2-star rating and a module with five hundred ratings averaging
3.4 are not the same problem, but a sorted list of averages presents them as if
they were. Modules need a minimum number of ratings before they can appear in a
"needs attention" list, and the view states how many were left out for being
too small to judge.

**Cost of this choice:** a genuinely bad new module stays invisible until it
collects enough ratings. The threshold is a lever, not a constant.

### Not using the unverified join key

Support desk B's `User Id` would visibly increase match rates if joined on. It
also might be a helpdesk-internal ID in a completely different namespace, in
which case joining would attach the wrong tickets to the wrong people —
silently, plausibly, and at scale.

The cost of not joining is measurable and visible: a lower coverage number. The
cost of joining wrongly is invisible until an agent acts on someone else's
support history. I took the visible cost.

**What would change my mind:** confirmation from the system owner that the ID is
the LMS learner ID. A single conversation resolves this — it's a question, not a
modelling problem.

### Raw layer stays untouched

Data lands exactly as it arrives. Every content column is `TEXT`, nothing is
cast or cleaned at ingestion, and every row carries `_source_file` and
`_loaded_at`. All correction happens in the `stg_*` views.

This costs storage and an extra layer. It buys the ability to answer "was this
value wrong in the source, or did we break it?" — which, in a system whose
entire job is establishing trust in consolidated data, is the question that
matters most.

### A new data source must match its export before anything reads it

When a source moves from file exports to an API, the API path is not trusted
because it runs. It is compared against an export of the same scope — row
counts, headline metrics, and individual answers — and only switched on when
they agree. Differences get explained (paging gaps, time zones, missing fields)
rather than accepted.

**Cost of this choice:** a slower switchover, and a dependency on someone
producing a matching export.

### Unexplained behaviour gets recorded, not smoothed over

Where something in a source system doesn't add up — a parameter the schema
doesn't declare, two columns that should agree and don't, a field whose meaning
nobody can confirm — the finding is written down as an open question rather than
resolved by assumption. Several `-- FLAG:` comments in the SQL exist for exactly
this reason.

It's the same instinct as the join-key decision above. An assumption that turns
out wrong is expensive precisely because nobody remembers making it.

## Roadmap

Six phases. The order follows one rule: nothing is built on a join or a data
path that hasn't been shown to be trustworthy yet.

1. **Ratings view, validated with users.** The single-source view described
   above, run on exported files. Timeboxed, and finished by putting it in front
   of two or three real users and asking what decision they would make from it
   — not by polishing it.
2. **API ingestion into the raw layer, on a schedule.** Replace file uploads
   with the reporting API, loaded into the database on a schedule rather than
   fetched live by the app. Content ratings first (checked against an export),
   then live session ratings and learner details. Loads replace a scope in one
   transaction, refuse suspiciously small or partial results, and log every run
   so a failed load raises an alert instead of silently serving stale data.
3. **More programmes, real hosting, access control.** Opening the tool beyond
   one portfolio means removing portfolio-specific assumptions (programme
   mappings, survey wording), hosting it on shared infrastructure, and deciding
   who can see which learners' data. This is where authentication stops being
   noise.
4. **Learner search and the consolidated profile.** The search-first home
   screen. A database lookup across attributed sources, with unmatched records
   shown. No language model required.
5. **Support desks.** Ticket data joined on email only, with the identifier
   questions below resolved or explicitly left open.
6. **Language model layer.** Themes and sentiment in comments and ticket text,
   natural-language questions over the data, and eventually at-risk alerts.
   Each item is processed once and the result stored, so cost scales with new
   data rather than page views.

**Running in parallel from the start:** the two identity questions — whether
support desk B's `User Id` is the LMS learner ID, and whether support desk A can
capture an identifier at ticket creation. Both are conversations with other
teams, not engineering tasks, and both take calendar time. Starting them in
phase 5 would stall phase 5.

## Deliberately not built (yet, or at all)

- **Sentiment scoring on ticket text** — phase 6. Scoring sentiment you can't
  attribute to a learner produces a number nobody can act on, so it waits until
  attribution is good enough.
- **Alerting on at-risk learners** — phase 6, same reason, plus alerting on a
  system with known coverage gaps trains people to ignore alerts.
- **Authentication and roles** — not in the prototype. Required before the tool
  is opened to other teams in phase 3, since the data is personal.
- **Write-back to source systems** — not planned. Turns a read-only lookup tool
  into something needing change management in four systems. Different product.
- **Real-time sync** — not planned. A scheduled batch is sufficient for both
  user moments above. Neither the agent nor the manager needs sub-hour
  freshness.

## How I'd know if this worked

Hypotheses, not results — the prototype runs on synthetic data and has no users.

| Measure | Why it matters | Current |
| --- | --- | --- |
| Attribution rate per source | Direct measure of the core problem. Already computed in `stg_source_coverage`. | Computed |
| Support desk A attribution | Moves off 0% only if the source system changes. The number that justifies that ask. | 0%, by construction |
| API vs export parity | A source isn't switched to the API until its numbers match a file export of the same scope. | Checked per source |
| Decisions named by users in phase 1 | If users can't name a decision the ratings view changes, the view is decoration. | Not measurable yet |
| Profile opens before a support reply | Does anyone use it mid-ticket, or is it a tool people admire and ignore? | Not measurable yet |
| Time-to-resolution, with context vs without | The claimed benefit. Needs real usage, and care to separate from ticket difficulty. | Not measurable yet |

The last one is the real proof and the hardest to get cleanly. I'd expect to
argue about its confounds before trusting it.

## What running this for real would take

Money is the small part. The data volumes are modest by database standards, and
a schema on existing infrastructure plus a small hosting slot covers phases 2 to
5. The language model layer is the only meaningful running cost, and processing
each item once keeps it proportional to new data. Sending learner text to an
external model provider also needs security and legal sign-off before it is
built, not after.

The larger cost is ownership: alerts on failed loads, credential rotation,
source systems renaming fields, survey wording changing under the category
mappings, new programmes needing mapping checks, backups, and periodic access
reviews. A tool other teams rely on needs a named owner and a backup in
engineering, not one person's laptop.

## Architecture

```
reporting API  ──▶  scheduled  ──▶  raw_*   ──▶  stg_* views  ──▶  Streamlit app
file exports        loader          tables       (canonical)
(synthetic here)
```

**Raw layer.** Untouched landing tables, `TEXT` columns, indexed join keys,
`_source_file` and `_loaded_at` on every row.

**Staging layer.** `stg_*` views handle normalisation, type casting, and mapping
every source's field names onto one vocabulary — `learner_id`, `learner_email`,
`learner_name`, `program_id`, `cohort_id`, `course_id`, `ticket_id`. Every view
carries `is_attributable`, and unmatched rows carry a reason.
`stg_learner_identity` backs search; `stg_source_coverage` reports the health
metric.

**Application layer.** Queries views, never raw tables. Views rebuild with
`CREATE OR REPLACE` after each ingestion.

### Why ingestion is one interface, not six

The connector layer is modelled on a single parameterised reporting endpoint:
one query shape, the report selected by an identifier, and filters composed as a
conjunction of conditions, with paging handled by an offset and a row limit.
Under that pattern, adding a source is a configuration entry rather than a new
integration.

The synthetic sources were built to fit the same shape, so the CSV loader in
`scripts/load_raw.py` and any future API client satisfy one interface. Each
source is a dict entry naming its columns and target table; the reading
mechanics are shared. Swapping a file reader for a network client changes where
rows come from and nothing downstream of the raw tables.

This is the main reason the prototype isn't blocked on live access. Access
determines where raw rows originate. Everything above the raw layer — identity
resolution, attribution flags, coverage reporting, the interface — is built and
testable without it.

## Project status

- [x] Raw table DDL for all seven sources
- [x] `stg_*` staging views with canonical mapping and attribution flags
- [x] `stg_learner_identity` union view for lookup
- [x] Data-quality views — coverage and unattributable records
- [x] Synthetic data generator reproducing real-world identity defects
- [x] CSV loader with ingestion logging and positional-header guards
- [ ] Ratings view on synthetic data *(phase 1, in progress)*
- [ ] Scheduled loader with partial-load guards and run log *(phase 2)*
- [ ] Learner search and consolidated profile *(phase 4)*
- [ ] Screenshots and a walkthrough
- [ ] Theme and sentiment classification on feedback text *(phase 6)*

## Repository layout

```
├── sql/
│   ├── raw_tables_ddl.sql       # landing tables, ingestion log
│   └── staging_views.sql        # canonical mapping, identity, quality views
├── seed/
│   └── generate_fake_data.py    # synthetic source exports
├── scripts/
│   └── load_raw.py              # CSV -> raw tables
├── app/                         # Streamlit application
└── docs/
```

## Getting started

Requires Python 3.10+ and MySQL 8.0+.

```bash
git clone <repo-url> && cd pulse
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env              # fill in your connection details
```

Create the schema:

```bash
mysql -u <user> -p <database> < sql/raw_tables_ddl.sql
mysql -u <user> -p <database> < sql/staging_views.sql
```

Generate data and load it:

```bash
python seed/generate_fake_data.py --learners 500 --summary
python scripts/load_raw.py --dry-run      # verify the mapping
python scripts/load_raw.py --truncate     # load
```

The generator reproduces the identity defects described above on purpose —
missing emails, case variance, orphan tickets, a source with no identifier at
all. Clean test data would hide exactly the problems this tool exists to handle.
Pass `--clean` for defect-free output and `--seed N` for reproducible runs.

Then look at the headline numbers:

```sql
SELECT * FROM stg_source_coverage;
SELECT * FROM stg_unattributable_records;
```

## Stack

Streamlit · MySQL 8 · Python (pandas, SQLAlchemy)

## Licence

MIT — see [LICENSE](LICENSE).

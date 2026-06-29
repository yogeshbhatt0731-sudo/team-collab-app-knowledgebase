# MySQL — Interview Defense Sheet
### Team Collaboration App · Workspace Service · placement prep

**Golden rule:** your defense isn't a clever feature — it's judgment. You evaluated the alternative against *your own schema* and chose deliberately under a deadline. Everything below is anchored to your actual tables (`workspace`, `project`, `feature`, `sprint`, `task`, `comment`, and the junctions). If a follow-up goes deeper than you can defend, *"I'd have to check the internals to answer precisely"* is honest and costs you nothing. Bluffing costs you everything.

---

## TIER 1 — The decision (must know cold)

**Q: Why did you choose MySQL for the Workspace service?**
I'm deeply comfortable in it, it fits this schema without issue — it's standard relational data with foreign keys between workspace, project, sprint, task and comment — and with limited time before placements I'd rather demonstrate real depth in one stack than spread thin across two. It was a deliberate prioritization call, not a default.

**Q: Would PostgreSQL have been a better choice here?**
I looked at it specifically against this schema. Postgres offered two things that fit my design: a **partial index** for my backlog query — a backlog task is just a task with `sprint_id IS NULL`, so I query `WHERE project_id = ? AND sprint_id IS NULL` constantly — and **sequence-based IDs**, which let Hibernate batch bulk inserts. Both are real, but I'll be precise about how much they'd actually buy *this* app — and the honest answer is "not much at my scale," which is why staying in MySQL was the right tradeoff. *(Full breakdown in the performance section below — know it, it's your sharpest answer.)*
*↳ "I considered the alternative, measured what it offered against my schema, and still chose this" sounds like someone who makes decisions, not someone who picks defaults.*

**Q: What's the difference between MySQL and PostgreSQL?**
Both are mature open-source relational databases. MySQL historically prioritized simplicity and read performance and is one of the most widely deployed databases anywhere. Postgres leans toward standards-compliance and a richer feature set — partial indexes, stricter defaults, more advanced types. For a CRUD app like mine, either is a sound choice.

---

## TIER 2 — Fundamentals you should hold (asked regardless of DB)

**Q: What is ACID?**
Four guarantees a transaction gives. **Atomicity** — all-or-nothing. **Consistency** — valid state to valid state, constraints always hold. **Isolation** — concurrent transactions don't corrupt each other. **Durability** — once committed, data survives a crash. MySQL provides all four through its InnoDB engine.

**Q: What's a transaction — give an example from your app.**
A group of operations treated as one unit: all succeed or none do. In my app, "create a task and assign it to a user" writes to both `task` and `task_assignee` — that's one transaction, so I never end up with an assignee row pointing at a task that didn't save.

**Q: What's an index — where would you add one in your schema?**
A data structure (usually a B-tree) the database keeps next to a table so it can jump straight to matching rows instead of scanning all of them. In my schema I'd index the hot foreign-key lookups: `comment.task_id` (loading a task's comments), `task.sprint_id` and `task.feature_id` (grouping tasks), and a composite `(project_id, sprint_id)` on `task` for the backlog query. The tradeoff is slightly slower writes and extra disk.

**Q: Primary key vs foreign key — in your schema?**
A primary key uniquely identifies a row in its own table — `task_id` in `task`. A foreign key references a primary key in another table — `task.project_id` points at `project.project_id`, linking the task to its project. My junction tables like `task_assignee` use a *composite* primary key made of two foreign keys (`task_id`, `user_id`).

---

## TIER 3 — Project-specific DB questions (your schema invites these)

**Q: Why a database per service instead of one shared database?**
A shared database couples services through its schema — one team's migration can break another service. Private databases let each service deploy and migrate independently. The tradeoff I'll name up front: I lose cross-service SQL joins and foreign-key enforcement across boundaries — which is exactly what the next question is about.

**Q: Your `user_id` columns have no foreign key to a users table. Why?** *(your schema's signature question)*
Because `USER` lives in the Auth service's database, and you can't enforce a foreign key to a table your service can't see. So in the Workspace DB, every `user_id` — `created_by`, assignees, comment authors — is a bare *logical* reference, not an enforced FK. To show a name I'd call the Auth service or cache a lightweight projection. Right now, before Auth exists, the user arrives via an `X-User-Id` header, isolated to one extraction point so it's a one-file change when real auth lands.
*↳ The question most students with a paper microservices design fumble — they write JPA as if it were still one monolith with FK joins. Nailing it proves you understood the consequence of splitting auth out.*

**Q: Your `task` table has `project_id`, but a task's project is also reachable via its feature and sprint. Isn't that redundant?**
Deliberately, yes. The direct `task.project_id` makes the backlog query a simple indexed lookup with no joins, and lets a task with no feature or sprint still know its project. The tradeoff: the database doesn't enforce that the three agree, so my **service layer** validates that a task's `project_id` matches its feature's and sprint's project before saving. That's my business-layer validation — a stateful check the DTO can't do.

---

## Scope of performance improvement with PostgreSQL (for THIS project)

The honest, measured version — this is what makes you sound like an engineer instead of a brochure. Where Postgres would actually help my workload, and by how much:

| Workload in my app | What Postgres offers | Realistic gain at my scale | Verdict |
|---|---|---|---|
| Backlog query `task WHERE project_id=? AND sprint_id IS NULL` | Partial index over only backlog rows | MySQL already answers this well with a composite `(project_id, sprint_id)` index; Postgres's partial index is just smaller and cheaper to maintain | **Marginal** |
| Bulk **INSERT** (seeding tasks, importing a backlog, mass-creating comments) | Sequence pre-allocation → Hibernate batches many inserts into few round trips | MySQL's `AUTO_INCREMENT` forces ~one round trip per insert; 50 inserts ≈ 50 trips vs a handful on Postgres | **Real & measurable** — but only on bulk-insert paths |
| "Move tasks into a sprint" `UPDATE task SET sprint_id=?` | — nothing — | This is an **UPDATE**; Hibernate batches updates fine on MySQL too. The sequence advantage is **insert-only** | **No advantage** |
| Single-row CRUD (create one task, fetch a project, post a comment) | — nothing meaningful — | Both use B-tree indexes identically | **No difference** |
| Reads by foreign key (comments by task, tasks by feature/sprint) | — nothing — | Standard indexed lookups, equivalent on both | **No difference** |

**The verdict to say out loud:** *"On everyday operations — single-row CRUD and indexed reads — there's effectively no difference. The one genuine gap is bulk-insert throughput: MySQL's auto-increment stops Hibernate from batching inserts the way Postgres sequences allow. At my data scale that's negligible; it'd start to matter if I were importing thousands of tasks at once, which isn't this app's workload. So I gave up almost nothing in practice, and that's exactly why the tradeoff favored the stack I know."*

**The precise correction that earns points:** if they assume "pulling tasks into a sprint" would be slower on MySQL, gently correct it — that operation is an `UPDATE`, and updates batch fine on both. The insert-batching limitation is specifically about generated primary keys, which only inserts need. Knowing *that* distinction is the difference between memorizing a fact and understanding it.

---

## Traps to avoid (what NOT to say)

- ❌ Don't repeat outdated MySQL myths against yourself. If someone says *"MySQL doesn't enforce CHECK constraints,"* correct it: **MySQL 8.0.16+ enforces them.** Knowing where MySQL stands today is a credibility signal.
- ❌ Don't claim Postgres-only features (partial indexes, sequences, JSONB) as things *you're* using — you're not. Only mention them as what you *evaluated and passed on*.
- ❌ Don't oversell — "MySQL is more ACID" is wrong. InnoDB and Postgres are both fully ACID; they're peers here.
- ❌ Don't bluff on internals (MVCC, redo/undo logs, query optimizer). "I don't know that depth" is a perfectly good student answer.

---

## If you switch to Postgres later

Nothing in your architecture blocks it — driver + URL + a couple of config lines, because everything runs through JPA. The keep-the-door-open work is already done, so this decision is genuinely reversible. You've deferred it, not foreclosed it — the right move under a deadline.

---

### The ~1-hour prep checklist
- [ ] Tier 1 — all three answers, especially "would Postgres have been better" → routes into the performance section
- [ ] Performance section — can say the verdict line *and* land the UPDATE-vs-INSERT correction
- [ ] Tier 2 — ACID, transaction (your task/assignee example), index (your real indexes), PK vs FK (your composite keys)
- [ ] Tier 3 — the `user_id`-no-FK answer AND the `task.project_id` redundancy answer (both signature to your schema)
- [ ] Can correct the "MySQL doesn't do CHECK constraints" myth
- [ ] Time saved vs Postgres prep → redirect to architecture: bounded contexts, modular monolith, validation layers

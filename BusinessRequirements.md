# Business Requirements — Gamification for Sirma Academy LMS

## 1. Overview
Sirma Academy is a Learning Management System (LMS) platform offering courses, seminars, and structured learning paths (`Program`).

The purpose of this document is to define the business requirements for introducing a **gamification
layer** on top of the Sirma Academy platform — a set of mechanics aimed at increasing user engagement,
motivation, and course completion rates.

For each gamification feature described below, the
document outlines **what** it is, **why** it provides value, and **how** it fits into the existing domain
model.

### 1.1 Live vs. Self-Paced Learning
The platform serves two distinct delivery formats, both of which every gamification feature must
account for:

- **Live** — instructor-led sessions with a fixed schedule. This covers every `SeminarInstance` (each
  `curriculum` row carries `startDateTime`/`endDateTime`, `url` and `slidoCode`), as well as any
  `CourseInstance` with `isLive: true`.
- **Self-paced** — `CourseInstance` documents with `isLive: false`, where users progress through
  `curriculum[].lessons[]` on their own schedule and completion is tracked per-lesson via
  `lessons[].completedBy`.

### 1.2 Constraints & Extension Strategy
The existing Mongoose models (`Seminar`, `SeminarInstance`, `Course`, `CourseInstance`, `Program`,
`Survey`, `User`, `Certificate`) are **closed for modification** — no fields, schema changes, or
behavioral changes may be introduced into them. All gamification data and behavior described in this
document is implemented through **new, additional Mongoose collections** that reference the existing
models by `ObjectId` (e.g. a new model referencing `User` and `CourseInstance`), following the same
referencing pattern already used throughout the codebase. Where a feature needs to react to an
existing field (e.g. `lessons[].completedBy`, `Course.successfullyCompletedByUsers`), it does so by
reading that field, never by writing to or altering the existing schema.

## 2. Goals and Success Metrics

### 2.1 Goals
- Increase the completion rate of courses/seminars.
- Encourage regular, long-term use of the platform.
- Improve user retention across courses.

### 2.2 Success Metrics
- % growth in `successfullyCompletedCourses` relative to `enrolledCourseInstances` for each user.
- Average `Certificate.score` before/after introducing gamification.
- % of users enrolled in more than one `CourseInstance`/`SeminarInstance`.

## 3. Scope & Out of Scope

### 3.1 In Scope
- Live formats: `SeminarInstance`, and any `CourseInstance` with `isLive: true`.
- Self-paced formats: `CourseInstance` with `isLive: false`.
- All seven features listed in Section 6, implemented as new, additive collections referencing the existing models.

### 3.2 Out of Scope
- `Survey` this document does not define a points mechanic for filling in surveys.
- Payment-related fields on `CourseInstance` (`isPaid`, `price`, `paidUsers`) are unaffected — gamification rewards are never tied to payment status.

## 4. New Entities Overview
Every feature below is implemented as a new, additive Mongoose collection — none of the existing
models are modified. This table gives a single reference point for all new entities introduced; each BR links back to the relevant row(s).

| New Entity | Purpose | References |
|---|---|---|
| `PointsLedger` | Records each point-earning event (amount, reason, timestamp) | `User`, `CourseInstance`/`SeminarInstance` |
| `Attendance` | Tracks a user's presence/participation in a live session, since `SeminarInstance`/live `CourseInstance` have no such field | `User`, `CourseInstance`/`SeminarInstance`, curriculum lesson index |
| `Badge` | Defines an awardable badge (name, description, icon, criteria) | — (catalog entity) |
| `UserBadge` | Records a badge earned by a user, and when/where | `User`, `Badge`, `CourseInstance`/`SeminarInstance`/`Course` |
| `CertificateEnhancement` | Overlays visual customization (badges, theme) on an existing `Certificate` without altering it | `Certificate` |
| `Team` | Groups enrolled users for team-based competition within an instance | `User`, `CourseInstance`/`SeminarInstance` |
| `LiveEngagementSnapshot` | Captures real-time engagement data during a live lesson | `CourseInstance`/`SeminarInstance`, curriculum lesson index |
| `KnowledgeNode` | Represents a course unlocked by a user, forming a node in the Knowledge Map | `User`, `Course`, `CourseInstance` |
| `KnowledgeRelation` | Defines relationships between courses (prerequisite, related, advanced) for the Knowledge Map | `Course` |

## 5. Risks & Open Questions
- **Leaving an instance:** what happens to a user's points/badges/team membership if they are
  removed from `enrolledUsers` on a `CourseInstance`/`SeminarInstance`? (Proposal: retain historical
  points/badges as a record of achievement; remove only active team/leaderboard participation.)
- **Leaderboard visibility:** should the leaderboard respect `User.private`, which already exists on
  the model to mark a user's profile private? (Proposal: private users are excluded from public
  leaderboards by default.)
- **Duplicate point awards:** point-awarding events must be idempotent, similar to how
  `reminderSentAt`/`hourBeforeSentAt` already prevent duplicate reminder emails on
  `SeminarInstance`/`CourseInstance` — the same pattern should guard against double-counting
  attendance or completion events.
- **Live vs. self-paced parity:** not every feature naturally has a self-paced equivalent (e.g. the
  Live Session Energy Meter, BR-06). Section 6 states explicitly, per feature, whether a self-paced
  analogue exists or whether the feature is live-only by design.
- **Slido dependency:** the Energy Meter (BR-06) builds on the existing `slidoCode` field, implying a
  live polling integration; the exact data contract with Slido is an external dependency not owned by
  this document.

## 6. Feature Requirements

### 6.1 BR-01: Point System

**What**
Introduce **Sirma Points (SP)** — a single, platform-wide currency of engagement. Users earn SP for
concrete learning actions, and the running balance/history is stored in a new `PointsLedger` collection. SP is the foundation all other gamification features (Leaderboard,
Badges, Teams) build on.

Proposed SP-earning events:

| Event | Format | Trigger (existing model signal) | SP |
|---|---|---|---|
| Lesson completed | Self-paced | New entry appended to `CourseInstance.curriculum[].lessons[].completedBy` for the user | e.g. +10 SP |
| Session attended | Live | New `Attendance` record for the user on that lesson/session (see BR-06 dependency) | e.g. +15 SP |
| Course successfully completed | Both | User added to `Course.successfullyCompletedByUsers` | e.g. +50 SP |
| Certificate earned | Both | New `Certificate` document created for the user, referencing `courseInstance`/`course` | +25 SP, scaled by `Certificate.score` |

**Why**
- Gives every learning action an immediate, visible reward, which is the main lever for increasing
  the completion-rate and retention goals.
- A single SP currency (rather than per-feature counters) is what makes Leaderboard, Badges and Teams
  possible without inventing a new "value" concept for each of them — every later feature simply reads
  from the same `PointsLedger`.
- Rewarding both self-paced lesson completion and live attendance means neither delivery format is
  disadvantaged, directly addressing the live/self-paced requirement.

**How**
- New collection `PointsLedger`:
  ```js
  {
    user: { type: ObjectId, ref: 'User', required: true },
    amount: { type: Number, required: true },
    reason: { type: String, enum: ['LESSON_COMPLETED', 'SESSION_ATTENDED', 'COURSE_COMPLETED', 'CERTIFICATE_EARNED'], required: true },
    sourceType: { type: String, enum: ['CourseInstance', 'SeminarInstance', 'Course', 'Certificate'], required: true },
    sourceId: { type: ObjectId, required: true },
    createdAt: { type: Date, default: Date.now },
  }
  ```
- A user's SP balance is derived, not stored redundantly: `sum(PointsLedger.amount)` for that
  `user`. This avoids adding a `points` field to `User`, which would violate the "models are closed"
  constraint.
- Self-paced trigger: a background job/hook reacts to updates of `lessons[].completedBy` on
  `CourseInstance` and inserts a `PointsLedger` entry per new completion. Since `completedBy` already
  stores `completedAt`, the ledger entry is idempotent per `(user, lesson)` pair — one lesson can only
  award SP once.
- Live trigger: reacts to new `Attendance` records (BR-06 introduces this collection) instead of
  `completedBy`, since neither `SeminarInstance` nor a live `CourseInstance` lesson has a per-user
  completion marker.
- Idempotency follows the same pattern already used by `reminderSentAt`/`hourBeforeSentAt` on
  `SeminarInstance`/`CourseInstance`: each award-worthy event is checked against existing
  `PointsLedger` entries before a new one is written, so re-processing the same event never
  double-awards SP.

### 6.2 BR-02: Leaderboard

**What**
A ranked view of users by accumulated **Sirma Points (SP)**, shown at two scopes:
- **Instance leaderboard** — ranks only the users enrolled in a given `CourseInstance` or
  `SeminarInstance` (i.e. members of that instance's `enrolledUsers`), scoped to SP earned from that
  instance's events.
- **Global leaderboard** — ranks users by total SP across all instances.

**Why**
- Turns the otherwise invisible `PointsLedger` balance from BR-01 into a visible, comparative signal —
  competition and social proof are proven engagement drivers, directly supporting the "encourage
  regular use" and "improve retention" goals.
- An instance-scoped leaderboard matters for both formats: a live seminar's leaderboard creates
  in-session competitive energy (feeds into BR-06), while a self-paced course's leaderboard gives
  long-running motivation to users who don't share a live schedule with anyone.
- A global leaderboard rewards cross-course engagement, supporting the "% of users enrolled in more
  than one `CourseInstance`/`SeminarInstance`" metric.

**How**
- No new persistent collection is required beyond `PointsLedger` (BR-01) — the leaderboard is a
  read/aggregation view, not stored state:
  - Instance leaderboard: `PointsLedger` entries filtered by `sourceId` (and, transitively, cross-
    referenced against the `enrolledUsers` array of the relevant `CourseInstance`/`SeminarInstance`),
    grouped by `user`, summed, sorted descending.
  - Global leaderboard: same aggregation with no `sourceId` filter.
- Recomputed on read (or cached with a short TTL for performance on high-traffic instances) — this
  keeps the feature fully additive, since it never writes back to `CourseInstance`/`SeminarInstance`/
  `User`.
- Privacy: the existing `User.private` flag is honored — a private user's SP still counts toward
  totals but their name/entry is excluded from any leaderboard visible to other users (surfaced as
  "Anonymous" or simply omitted).
- Live vs. self-paced: both formats use the identical aggregation logic; the only difference is which
  `sourceId` values are included in the "instance leaderboard" filter (a `SeminarInstance` id vs. a
  `CourseInstance` id), so no separate implementation is needed per format.

### 6.3 BR-03: Badges

**What**
A **Badge** is a named, iconographic achievement a user unlocks by meeting a defined criterion.
Two new collections back this feature:
- `Badge` — the catalog entry (name, description, icon, criterion type, `hidden` flag).
- `UserBadge` — the record of a specific user earning a specific badge, and where/when.

Badges come in two flavors:
- **Regular** — visible in the catalog at all times (locked/unlocked state shown), so users know what
  to aim for.
- **Hidden (`Badge.hidden: true`)** — not shown in the catalog until unlocked; discovered by surprise,
  which rewards exploration/curiosity rather than a known checklist.

**Example badge catalog**

| Badge | Type | Criterion (existing model signal) | Format |
|---|---|---|---|
| First Steps | Regular | First entry in user's `PointsLedger` | Both |
| Course Finisher | Regular | User added to `Course.successfullyCompletedByUsers` | Both |
| Perfectionist | Regular | `Certificate.score === 100` for the user | Both |
| Polyglot Learner | Regular | User enrolled in `CourseInstance`/`SeminarInstance` with 3+ different `category` values | Both |
| Marathoner | Regular | 5 lessons completed (`completedBy` entries) in a single calendar day | Self-paced |
| Front Row Fan | Regular | Attended (via `Attendance`) every session of a `SeminarInstance`/live `CourseInstance` from its first `curriculum` row to its last | Live |
| Night Owl | Hidden | `Attendance`/`completedBy.completedAt` timestamp between 00:00–04:00 local time | Both |
| Early Bird | Hidden | Lesson/session completed or attended more than 24h before `startDateTime` (self-paced) or joined a live session before its scheduled start | Both |
| Comeback Kid | Hidden | User completes a lesson after a 30+ day gap since their previous `PointsLedger` entry | Self-paced |
| Lucky Number 7 | Hidden | Exactly the 7th user to join a `Team` (BR-05) or the 7th to complete a given `Course` | Both |

**Why**
- Regular badges give users a visible, structured goal list, reinforcing the completion-rate goal in a 
  way SP alone (a single number) doesn't — badges communicate what kind of engagement
  is valued, not just how much.
- Hidden badges add delight and replay value: they reward users who go beyond the minimum path (e.g.
  attending early, returning after a break), which supports the "long-term use" and retention goals
  without requiring the platform to expose every mechanic up front.
- Splitting badges across both formats (rather than only self-paced) again respects the live/self-paced
  requirement — e.g. "Front Row Fan" exists specifically because live sessions have no
  other progress marker besides attendance.

**How**
- New collection `Badge`:
  ```js
  {
    key: { type: String, required: true, unique: true },
    name: String,
    description: String,
    icon: String,
    hidden: { type: Boolean, default: false },
    criterionType: { type: String, enum: ['POINTS_EVENT', 'COURSE_COMPLETION', 'CERTIFICATE_SCORE', 'ATTENDANCE_PATTERN', 'CUSTOM'] },
  }
  ```

- New collection `UserBadge`:
  ```js
  {
    user: { type: ObjectId, ref: 'User', required: true },
    badge: { type: ObjectId, ref: 'Badge', required: true },
    earnedAt: { type: Date, default: Date.now },
    sourceType: { type: String, enum: ['CourseInstance', 'SeminarInstance', 'Course'] },
    sourceId: { type: ObjectId },
  }
  ```
- Criteria are evaluated by listening to the same events that already drive BR-01 (new `PointsLedger`
  entries, new `Attendance` records, additions to `Course.successfullyCompletedByUsers`, new
  `Certificate` documents) — no new triggers are introduced, badges are a secondary consumer of events
  BR-01 already reacts to.
- A badge is awarded at most once per `(user, badge)` pair — enforced via a unique compound index on
  `UserBadge` (`user` + `badge`), the same idempotency principle used throughout this document.
- The catalog UI reads `Badge.hidden` to decide whether to render a locked placeholder (regular) or
  omit the entry entirely (hidden) for badges the user hasn't earned yet.

### 6.4 BR-04: Customizable Certificates

**What**
Allow a user's `Certificate` to be visually customized — e.g. a themed border, a colored seal, or one
or more badge icons displayed on the printed/rendered certificate — where the specific customization
is **unlocked by earning a `Badge`** (BR-03). The underlying `Certificate` document itself
(`certificateId`, `issueDate`, `score`, `user`, `courseInstance`, `course`) is never modified; the
customization lives entirely in a new, separate `CertificateEnhancement` collection that the
certificate renderer reads in addition to the existing `Certificate` fields.

Example mapping (badge → certificate customization):

| Badge earned | Certificate customization unlocked |
|---|---|
| Perfectionist | Gold foil border/theme + "Perfect Score" seal |
| Course Finisher | Badge icon row showing the course's completion badge |
| Front Row Fan | "Full Attendance" ribbon on certificates from live instances |
| Marathoner | Special "Fast Track" theme variant |

A user with no relevant badges still gets a fully valid, standard-looking certificate — customization
is additive decoration, never a requirement to receive the certificate itself.

**Why**
- Makes badges (BR-03) tangible outside the platform: a certificate is the one gamification artifact
  users routinely download, print, and share (e.g. on LinkedIn), so surfacing badges there extends
  their motivational effect beyond the in-app experience.
- Directly ties into the existing success metric "Average `Certificate.score` before/after
  gamification" — if better performance (e.g. `score === 100`) visibly upgrades the
  certificate's appearance, it creates a direct incentive loop between performance and this metric.
- Because it's implemented as an overlay rather than a schema change, it satisfies the "models are
  closed" constraint while still meeting the requirement to customize certificates "e.g.
  with Badges."
- Applies to both formats identically: `Certificate` already references either a `courseInstance` or
  a `course`, with no live/self-paced distinction needed at the certificate level, so the same overlay
  mechanism works for a live seminar's certificate and a self-paced course's certificate alike.

**How**
- New collection `CertificateEnhancement`:
  ```js
  {
    certificate: { type: ObjectId, ref: 'Certificate', required: true, unique: true },
    theme: { type: String, default: 'default' },
    displayedBadges: [{ type: ObjectId, ref: 'Badge' }],
    ribbon: { type: String, default: null },
  }
  ```
- Generation flow: when a `Certificate` is issued, a job checks the user's `UserBadge` entries (BR-03)
  relevant to that `course`, maps them to the table above, and writes (or updates) the
  corresponding `CertificateEnhancement` document — the `Certificate` document itself is only ever read,
  never written to.
- The certificate rendering step (PDF/image generation) is extended to look up
  `CertificateEnhancement` by `certificate` id and merge its `theme`/`displayedBadges`/`ribbon` into the
  existing template; if no `CertificateEnhancement` exists, the renderer falls back to the current,
  unmodified certificate design — so this feature is fully backward-compatible with certificates issued
  before gamification existed.
- Because `CertificateEnhancement.certificate` is unique, re-running the generation job is idempotent
  (upsert), consistent with the idempotency principle used across BR-01/BR-03.

### 6.5 BR-05: Teams

**What**
A **Team** groups a subset of users enrolled in the same `CourseInstance`/`SeminarInstance` into a
shared unit for collective competition. A team has a name, its member list (drawn only from that
instance's `enrolledUsers`), and an aggregate SP score derived from its members' `PointsLedger`
entries earned within that instance. Teams appear on a dedicated **team leaderboard**, alongside (not
replacing) the individual leaderboard from BR-02.

**Why**
- Individual competition (BR-02) doesn't work equally well for everyone — some users are motivated by
  group belonging rather than personal ranking. Teams give a second, complementary motivational path,
  supporting the same retention/engagement goals through a different mechanic.
- For live formats especially, teams create natural in-session dynamics (e.g. teachers can call out
  team standings during a seminar, feeding into the Energy Meter atmosphere in BR-06).
- For self-paced formats, teams give users a reason to coordinate progress with peers on a course
  they'd otherwise complete in isolation, which can reduce drop-off — directly supporting the
  completion-rate goal.
- Scoping teams to a single instance (rather than platform-wide) keeps the mechanic simple and
  meaningful: members share the same course/seminar content, so a team's combined score is always a
  fair, apples-to-apples comparison.

**How**
- New collection `Team`:
  ```js
  {
    name: { type: String, required: true },
    sourceType: { type: String, enum: ['CourseInstance', 'SeminarInstance'], required: true },
    sourceId: { type: ObjectId, required: true },
    members: [{ type: ObjectId, ref: 'User' }],
    createdAt: { type: Date, default: Date.now },
  }
  ```
- **Membership constraint:** a user may only be added to `Team.members` if they already appear in the
  referenced instance's `enrolledUsers` array — enforced at the application layer when a team is
  created or joined, not by modifying `CourseInstance`/`SeminarInstance` itself.
- **Team score** is derived, not stored: sum of `PointsLedger.amount` for all `user`s in
  `Team.members`, filtered to `sourceId` matching the team's instance — reusing the exact aggregation
  approach from BR-02 rather than introducing a new scoring mechanism.
- **Team leaderboard** is simply the BR-02 aggregation grouped by `Team` instead of by individual
  `user`, so it shares the same read/aggregation-only implementation (no extra write-path, no
  duplicated logic).
- If a user leaves an instance (removed from `enrolledUsers`), the application layer also removes them
  from any `Team.members` for that instance — consistent with the "leaving an instance" open question, 
  while their already-earned SP/badges remain untouched in `PointsLedger`/`UserBadge`.
- Live vs. self-paced: identical data model and scoring; the only practical difference is *when* team
  standings are surfaced — live instances can display them in real time during a session (tying into
  BR-06), while self-paced instances show them on the course dashboard.

### 6.6 BR-06: Live Session Energy Meter

**What**
A real-time visual indicator — an "energy meter" — shown to the teacher (and optionally participants)
during a live lecture, reflecting how engaged the audience currently is. It aggregates signals from
the current `curriculum` row of a `SeminarInstance` or a live `CourseInstance`
(`isLive: true`):  join/attendance events, Slido interaction activity (the existing `slidoCode` field
already implies a live polling integration for that lesson), and, where available, live SP-earning
events from BR-01. This is the collection `LiveEngagementSnapshot`, and it
is also the source of the new `Attendance` records that BR-01 and BR-05 depend on.

**Why**
- Live sessions are the one format where engagement is otherwise invisible to the teacher in real
  time — a self-paced course has `completedBy` timestamps to review after the fact, but a teacher
  mid-lecture has no equivalent feedback loop. The Energy Meter fills exactly that gap.
- Gives teachers actionable, in-the-moment information (e.g. "energy is dropping — good time for a
  Slido poll or a break"), which supports the completion/retention goals indirectly by improving the
  live session experience itself rather than only rewarding it after the fact.
- Because attendance is what feeds SP (BR-01), badges like "Front Row Fan" (BR-03), and team standings
  (BR-05), the Energy Meter's underlying `Attendance` data is the single source of live-engagement
  truth that the rest of the gamification layer depends on — it is not a standalone display gimmick.
- This feature is deliberately live-only by design: self-paced learners already have an equivalent 
  motivational signal in their personal `PointsLedger`/badge progress, so no artificial "self-paced energy meter" is introduced.

**How**
- New collection `Attendance`:
  ```js
  {
    user: { type: ObjectId, ref: 'User', required: true },
    sourceType: { type: String, enum: ['CourseInstance', 'SeminarInstance'], required: true },
    sourceId: { type: ObjectId, required: true },
    lessonIndex: { type: Number, required: true },
    joinedAt: { type: Date, default: Date.now },
    leftAt: { type: Date, default: null },
  }
  ```
- New collection `LiveEngagementSnapshot`:
  ```js
  {
    sourceType: { type: String, enum: ['CourseInstance', 'SeminarInstance'], required: true },
    sourceId: { type: ObjectId, required: true },
    lessonIndex: { type: Number, required: true },
    timestamp: { type: Date, default: Date.now },
    activeAttendees: { type: Number, required: true },
    slidoInteractionRate: { type: Number, default: null },
    energyScore: { type: Number, required: true },
  }
  ```
- A periodic job (e.g. every 10–30 seconds during a live lesson, identified by matching the current
  time against that `curriculum` row's `startDateTime`/`endDateTime`) computes `activeAttendees` from
  open `Attendance` records and, where available, pulls interaction data via the existing `slidoCode`
  for that lesson, then writes a new `LiveEngagementSnapshot`. The teacher-facing UI polls/subscribes
  to the latest snapshot for the session in progress.
- The Slido interaction data is an external dependency: the exact metric/API contract
  Slido exposes for a given `slidoCode` is outside this document's control and needs confirming with
  that integration before implementation.
- Because both `SeminarInstance` and a live `CourseInstance` share the same
  `curriculum[].startDateTime/endDateTime/slidoCode` shape, the same `Attendance`/
  `LiveEngagementSnapshot` logic serves both without any format-specific branching.

### 6.7 BR-07: Knowledge Map

**What**
A **Knowledge Map** is an interactive graph visualization representing a user's learning journey. Each successfully completed `Course` is represented as a node in the graph. Relationships between courses (for example prerequisite, continuation or related topics) are represented as edges connecting those nodes. Unlike the existing course catalog, which is primarily list-based, the Knowledge Map presents the user's accumulated knowledge as a connected network similar to the graph view popularized by Obsidian. Completed courses are visually highlighted, while courses not yet completed remain dimmed but visible, allowing users to understand both their progress and potential future learning paths. The feature applies equally to both live and self-paced learning formats because it derives its data from successful course completion rather than lesson progression.

**Why**

- Gives users a visual representation of their learning journey instead of a simple list of completed courses.
- Encourages users to continue learning by revealing nearby courses that naturally extend their current knowledge.
- Makes progress immediately visible, reinforcing intrinsic motivation alongside the competitive mechanics introduced by Points, Leaderboards and Badges.
- Helps identify knowledge gaps by displaying prerequisite relationships between courses.
- For users enrolled in a `Program`, the map also surfaces a concrete "what's next" recommendation
  rather than only a generic list of related courses — turning the graph from a pure recap into a
  lightweight discovery surface as well.

**How**

- New collection `KnowledgeNode`:
```js
{
    user: { type: ObjectId, ref: 'User', required: true },
    course: { type: ObjectId, ref: 'Course', required: true },
    courseInstance: { type: ObjectId, ref: 'CourseInstance' },
    unlockedAt: { type: Date, default: Date.now }
}
```
- New collection `KnowledgeRelation`:
```js

{
    fromCourse: { type: ObjectId, ref: 'Course', required: true },
    toCourse: { type: ObjectId, ref: 'Course', required: true },
    relationType: { type: String, enum: ['PREREQUISITE', 'RELATED', 'ADVANCED','RECOMMENDED']
    }
}
```

- `KnowledgeRelation` is a catalog entity defining how courses are connected. These relationships are maintained by content managers and are independent of any individual user.

- `KnowledgeNode` represents a course unlocked by a particular user. A new document is created when the user is successfully added to `Course.successfullyCompletedByUsers`, reusing the same trigger already used by BR-01 (Points) and BR-03 (Badges).

- The graph itself is derived rather than stored. When the user opens the Knowledge Map, the application loads all `KnowledgeNode` documents belonging to that user together with every matching `KnowledgeRelation` and constructs the graph dynamically.

- Program-aware "next step" overlay: for a user enrolled in a `Program` (`User.enrolledPrograms`), the
  map additionally calls the existing `Program.getNextCourse(userCompletedCourses)` method — passing the
  user's `successfullyCompletedCourses` — to identify the single next course the `Program` recommends.
  That course's node is rendered in a distinct "recommended" visual state, separate from the general
  "locked but related" nodes coming from `KnowledgeRelation`. This reuses `Program`'s own prerequisite
  logic (`Program.courses[].prerequisites`) exactly as written, without re-implementing or duplicating it
  in the gamification layer — the Knowledge Map simply visualizes a result the closed model already
  computes for us.
- Where a user is enrolled in more than one `Program`, `getNextCourse()` is called once per program, and
  each result is overlaid independently — a user can see multiple "next step" recommendations if they are
  pursuing several learning paths in parallel.
- Because `KnowledgeRelation` (general catalog relations) and the `Program`-derived "next step" overlay are
  two independent data sources feeding the same visual graph, there is no conflict or duplication risk
  even if a `Program`'s prerequisite structure and a `KnowledgeRelation` entry happen to describe the same
  pair of courses differently — the UI simply layers both signals (e.g. a dashed "related" edge vs. a
  highlighted "recommended next" node).

- Because the graph is reconstructed from existing collections, no additional state is duplicated and no existing models require modification, satisfying the "models are closed" constraint.


- A completed course is rendered as an active node, while related but incomplete courses remain visible in a locked state, encouraging users to continue progressing through adjacent learning paths.

- Each node may display additional information derived from existing gamification features, including earned badges (BR-03), certificate achievements (BR-04) and the user's accumulated Sirma Points (BR-01), without introducing any new dependencies between those features.

- The visualization is implemented as an interactive force-directed graph, allowing users to zoom, pan, drag nodes and navigate directly to a course by selecting its corresponding node.

- Since the graph is generated from existing completion events, the feature is naturally idempotent. Completing the same course multiple times never creates duplicate `KnowledgeNode` entries because uniqueness is enforced on the `(user, course)` pair.
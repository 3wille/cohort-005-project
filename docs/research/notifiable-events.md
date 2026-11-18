# Notifiable events in Cadence

Research for [#3 Inventory notifiable events in Cadence](https://github.com/3wille/cohort-005-project/issues/3), part of the [Notifications spec map (#2)](https://github.com/3wille/cohort-005-project/issues/2).

**Question:** Which domain events already happen in Cadence that could notify someone? For each, what triggers it, which role(s) it concerns, and is it time-sensitive or actionable?

**Sources:** the code under `app/`, `GLOSSARY.md` and `docs/adr/`, at commit `979e285` (local `main`). Every `file:line` below refers to that commit. Vocabulary follows `GLOSSARY.md`. The code's role names map to the glossary like this:

- `UserRole.Admin` is a **Platform Admin**.
- `TeamMemberRole.Admin` (in `team_members`) is a **Team Admin**.
- A coupon is a **Seat**.
- A row in `refunds` is a **Refund Request**.

Labels used in the tables:

- **Actionable**: the recipient is expected to do something.
- **Time-sensitive**: the event loses value, or causes harm, if the recipient learns about it late.

## What already notifies today

Only one channel exists: transactional email through Postmark. `app/lib/mailer.server.ts:21` handles delivery. Without a `POSTMARK_SERVER_TOKEN`, messages are written to `.outbox/` instead (`mailer.server.ts:49`, `:62-65`).

Three emails are sent, and all of them go to the **buyer** of the Purchase:

| Email | Built at | Sent at |
| --- | --- | --- |
| "Refund requested — {course}" | `app/lib/refunds.server.ts:393-405` | `app/routes/courses.$slug.refund.ts:75` |
| "Refund approved — {course}" | `app/lib/refunds.server.ts:736-780` | `app/routes/admin.refunds.tsx:130` |
| "Refund declined — {course}", with the Platform Admin's note | `app/lib/refunds.server.ts:736-780` | `app/routes/admin.refunds.tsx:130` |

There is no in-app notification of any kind. The individual-purchase flow has a stub where a welcome email would go (`app/routes/courses.$slug.purchase.tsx:173-175`, `// Would send welcome email here`), but `enrollUser` is called with `sendEmail = false` (`:371`).

## Implemented events

### Refunds

| Event | Trigger | Concerns | Time-sensitive / actionable |
| --- | --- | --- | --- |
| **Refund Request raised** (becomes an Open Refund Request) | `courses.$slug.refund.ts:63` → `raiseRefundRequest`, `refunds.server.ts:357` (insert `:380`). Posted from the course page (`courses.$slug.tsx:906`) for an Individual Purchase, or from the team page (`team.tsx:413`) for a Team Licence. | **Platform Admin**: must decide it. Today nobody tells them; they have to visit `/admin/refunds`. **Buyer** (a Student, or a Team Admin for a Team Licence): confirmation, already emailed. | **Platform Admin: actionable, time-sensitive.** While a Team Licence's request is open, its Unclaimed Seats are frozen (ADR 0003, `refunds.server.ts:111-120`). Buyer: informational. |
| **Refund Request approved** | `admin.refunds.tsx:111` → `approveRefundRequest`, `refunds.server.ts:614`. Individual Purchase: deletes the Enrollment unless the buyer also holds a Claimed Seat (`:645-666`, ADR 0004). Team Licence: marks Unclaimed Seats `refundedAt` (`:638-643`). | **Buyer**: already emailed. **Instructor**: a Student on their course becomes Lapsed (`app/lib/enrollments.ts`), and revenue drops. | Buyer: **time-sensitive**, because the Enrollment ends at approval, not when the money moves (ADR 0002). Instructor: informational. |
| **Refund Request declined** | `admin.refunds.tsx:118` → `declineRefundRequest`, `refunds.server.ts:692`. A note is required (`:701-707`). | **Buyer**: already emailed, with the note. For a Team Licence, the frozen Seats work again, so Students waiting to claim one may care. | Buyer: **actionable** (they may raise a new Refund Request, glossary *Declined Refund Request*). |

### Purchases, Seats and Enrollment

| Event | Trigger | Concerns | Time-sensitive / actionable |
| --- | --- | --- | --- |
| **Individual Purchase made, Enrollment created** | `courses.$slug.purchase.tsx:370-371` | **Student**: receipt and welcome (stubbed, `:174`). **Instructor**: a new Enrollment on their course. | Student: informational, but it starts the 14-day refund window (`app/lib/refund-rules.ts:5`). Instructor: informational, good candidate for batching. |
| **Team Licence bought, Seats created** | `courses.$slug.purchase.tsx:378` → `createTeamPurchase`. The team and its Team Admin membership are created on the first purchase (`:210-219`); Seats are created in `generateCoupons` (`:222`). | **Team Admin**: receipt, plus Seats to hand out. **Instructor**: a sale. | Team Admin: **actionable** (distribute the Seats). Instructor: informational. |
| **Seat claimed** | `redeem.$code.tsx:201` → `redeemCoupon`. Marks the Seat claimed (`:98`) and creates the Enrollment (`:107`). | **Team Admin**: a Seat was used. A Claimed Seat also makes the whole Team Licence non-refundable (glossary *Claimed Seat*, ADR 0003). **Student** who claimed: welcome. **Instructor**: a new Enrollment. | Team Admin: informational, but it matters if they were weighing a refund. Student and Instructor: informational. |
| **Seat claim refused** (on hold or refunded) | `redeem.$code.tsx:61`, `:153` via `getSeatAvailability`, `refunds.server.ts:111` | **Student** trying to claim. **Team Admin**, who may not know a colleague is blocked. | Student: shown inline, no notification needed. A "Seats available again" notice after a decline would be new behaviour. |

### Lessons, questions and quizzes

| Event | Trigger | Concerns | Time-sensitive / actionable |
| --- | --- | --- | --- |
| **Question asked** (top-level comment on a lesson) | `courses.$slug.lessons.$lessonId.tsx:903` → `createComment` `:404` with `parentId = null` | **Instructor** who owns the course. **Platform Admin**: the unanswered-questions queue spans every course for them (`instructor.questions.tsx:175-176`). Questions written by the owning Instructor or a Platform Admin don't enter the queue (`:63-70`). | Instructor: **actionable**. It becomes **time-sensitive** after 3 days, when the queue flags the question as stale (`instructor.questions.tsx:160`). |
| **Reply posted** (to a question or comment) | `courses.$slug.lessons.$lessonId.tsx:903` with a `parentId`. Staff answers from the queue go through `instructor.questions.tsx:211`. | **Student** who asked: their question was answered. Other repliers in the thread. If a Student replies, the owning Instructor may also want to know. | Asker: **actionable** (read the answer, follow up). It is the counterpart of "Question asked". |
| **Comment edited / deleted** | `courses.$slug.lessons.$lessonId.tsx:923`, `:935`. Staff (the owning Instructor or a Platform Admin) can delete other people's comments (`deleteComment` `:459-466`, `isStaff`). | **Comment author**, only when staff deleted it (moderation). | Low priority, informational. |
| **Quiz attempted** | `courses.$slug.lessons.$lessonId.tsx:970` → `computeResult` (attempt insert `:603`; passed if score > 0.7, `:599`) | **Student**: the result is shown immediately, in the same request. **Instructor**: aggregate interest only. | Not time-sensitive. A per-attempt notification adds little. |
| **Lesson marked complete** | `courses.$slug.lessons.$lessonId.tsx:948` | **Student** only. | Not notifiable on its own. See *Course completed* below. |

### Courses and users

| Event | Trigger | Concerns | Time-sensitive / actionable |
| --- | --- | --- | --- |
| **Course rated** (new rating or changed rating) | `courses.$slug.tsx:478` → `rateCourse` (update or insert) | **Instructor** | Informational, good candidate for batching. |
| **Course status changed** (Draft / Published / Archived) | By the Instructor: `instructor.$courseId.tsx:695`. By a Platform Admin, on any course: `admin.courses.tsx:142`. | **Instructor**, when a Platform Admin changes their course. **Enrolled Students**, when a course is archived. | Instructor: **actionable** when a Platform Admin unpublishes their course. |
| **Lesson added / deleted** | `instructor.$courseId.tsx:753`, `:827` | **Enrolled Students** ("new content", feature idea 29 in `_internal/feature-ideas.md`) | Informational, should be batched. |
| **Quiz re-saved** | `instructor.$courseId.lessons.$lessonId.quiz.tsx:399-403`. Saving deletes the existing quiz **and every Student's attempts and answers** (`deleteQuiz` `:201-220`). | **Students** who had attempts | Today it is silent data loss. Worth noting, even if the fix belongs outside notifications. |
| **Course price changed** | `instructor.$courseId.tsx:700` | Nobody yet (no wishlist or follow feature) | Not notifiable today. |
| **User role or profile changed by a Platform Admin** | `admin.users.tsx:128` (role), `:122` (name and email) | The **user** affected, e.g. promoted to Instructor or had their email changed | An email change is **security-relevant**. A promotion to Instructor is actionable (they can now create courses). |
| **Signed up** | `signup.tsx:78` (always creates a Student) | **Student**: welcome | Informational. |
| **Course created** (as a Draft) | `instructor.new.tsx:149` | Possibly **Platform Admin** (review), but no review flow exists | Not notifiable today. |
| **Video watch progress** | `api.video-tracking.ts:30` | Not notifiable on its own. It feeds *Watched* and the Watch Limit. | See *Implied events*. |

## Implied by the glossary or ADRs, not implemented

| Event | Source | Would concern | Notes |
| --- | --- | --- | --- |
| **Pending Refund** | Glossary *Pending Refund* | Buyer | Not implemented. `refunds` has no money-movement state; `amountRefunded` is set at approval (`refunds.server.ts:672`). Purchases are mock (no payment provider). |
| **Issued Refund** | Glossary *Issued Refund* | Buyer | Not implemented. The approval email already says the money "is on its way back to you" (`refunds.server.ts:759-760`), although nothing tracks or confirms that it was sent. |
| **Failed Refund** | Glossary *Failed Refund* | **Platform Admin** (actionable, time-sensitive) and the buyer | Not implemented. The Refund Request stays approved, and per ADR 0002 the Enrollment is already gone. |
| **Seats available again** (after a decline) | ADR 0003 | **Team Admin**, Students who were refused | Not implemented. The state changes implicitly when the request stops being Open. |
| **Refund window closing** | 14-day window, `refund-rules.ts:5` | Buyer | Not implemented. It would need a schedule, not a trigger. The window is counted in calendar days (`refunds.server.ts:46-56`). |
| **Approaching the Watch Limit** (refund eligibility ends) | Glossary *Watch Limit*; 1 hour, `refund-rules.ts:6` (also 50% of lessons completed, `:7`) | Student, if anyone | Not implemented. Derived from video tracking. Probably not worth notifying. |
| **Student became Lapsed** | Glossary *Lapsed*, `enrollments.ts` | Instructor | Happens implicitly on refund approval. See *Refund Request approved*. |
| **Course completed** | `enrollments.completedAt` column (schema) | Student (congratulations), Instructor | Not implemented. Nothing in `app/` ever writes `enrollments.completedAt`, so completion is never recorded. |
| **Unanswered question gone stale** | `instructor.questions.tsx:160` (3 days) | Instructor, Platform Admin | Only a visual flag in the queue. A reminder would need a schedule. |
| **Course announcement** | `_internal/feature-ideas.md` item 1 | Enrolled Students | Not implemented. |

## Findings that bear on "Which events notify which role"

1. **The most actionable event notifies nobody who has to act.** Raising a Refund Request emails only the buyer. The Platform Admin who must decide it finds out only by opening `/admin/refunds`. For a Team Licence, Seats stay frozen until they do.
2. **Recipients are relationships, not user roles.** `users.role` only knows `student`, `instructor` and `admin`. A Team Admin is a Student with a `team_members` row. "The Instructor" means `courses.instructorId`. "The buyer" means `purchases.userId`, who is a Team Admin for a Team Licence. Routing has to work out recipients per event (course owner, buyer, team admin, question asker), not from the user's role.
3. **Platform Admins already share the Instructor's question queue** (`instructor.questions.tsx:175-176`) across every course. Decide explicitly whether they receive question notifications, or only see the queue.
4. **An email vendor is already in place.** Postmark is used for refund emails, with a file-based `.outbox/` for local development. The vendor decision has to replace it or sit alongside it.
5. **Email is sent after the database write, with no retry or outbox.** At `admin.refunds.tsx:130` and `courses.$slug.refund.ts:75`, a Postmark failure throws after the decision has already been committed. The user sees an error, and a retry then gets a 409. Any notification layer should be triggered after the commit and tolerate delivery failure.
6. **The approval email over-promises.** It says the money is on its way, but Pending, Issued and Failed Refunds don't exist yet. The Failed Refund case, which needs a Platform Admin to act, has no trigger to hang a notification on.
7. **Several worthwhile notifications need a scheduler, not an action hook:** a question gone stale, the refund window closing, digests. Cadence has no job runner today. That favours a vendor with delays or digests built in, or a decision to leave these out of the first release.

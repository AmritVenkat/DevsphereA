# DevSphere — Feature Summary

Core purpose: automate advisor busywork (verification, chasing, reporting) and consolidate scattered WhatsApp/email coordination into one system. No surveillance features — nothing that adds monitoring work instead of removing it.

---

## Student features

- Profile view/edit — avatar, GitHub/LinkedIn/portfolio links, LeetCode username
- View own class group and assigned advisors (read-only — advisors are admin-assigned)
- Add/edit/delete projects (title, description, tech stack, links) — submitted for verification
- Log hackathon participation (name, date, result, certificate) — submitted for verification
- Upload/download certificates — submitted for verification
- Log achievements/milestones — self-logged, no verification needed
- Track verified/pending/rejected status on all submissions, with rejection reason if applicable
- Respond to advisor polls
- View announcements (all / student-targeted / advisor-specific scope)
- View admin-published hackathon postings, fill dynamic registration forms
- Notifications (hackathon posted, verification update, reminders, announcements)
- Download own verified portfolio as PDF (auto-generated resume-style export)

## Advisor features

- View assigned class group(s) — a class group can have multiple advisors, each for a different subject
- Bulk verify/reject student submissions (projects, hackathons, certificates)
- Verification templates — pre-set rejection reasons instead of retyping
- View verification history (audit trail) per student
- Create polls scoped to their class group, view responses + auto-summary at close
- Create announcements (all students, or scoped to their own class group)
- Download batch report (PDF) — verified achievement summary for their class group
- Download hackathon form responses (CSV) for their class group's students

## Admin features

- Create class groups (department, batch)
- Assign advisors to class groups (many-to-many, per subject)
- Create student/advisor accounts (via Supabase Admin API)
- Bulk advisor reassignment
- Publish hackathon postings with custom dynamic data-collection forms
- Create/schedule announcements (institution-wide or role-targeted)
- Global search across students/advisors/projects/certs
- Audit log (who verified/created/deleted what, when)
- Auto-escalation inbox — items unactioned past threshold surface here
- Institution-wide PDF report (aggregate verified achievements, advisor-wise submission volume — explicitly NOT turnaround-time tracking, to avoid surveillance framing)
- CSV export/import support

## Cross-cutting / automation features (the core differentiators)

- **Auto-reminder system** — automatic in-app + email reminders for pending verifications/poll responses, replacing manual advisor chasing
- **Auto-escalation** — unactioned items past a threshold auto-flag to admin, so advisors don't get chased either
- **Single notification pipe** — one system for hackathon postings, verification updates, reminders, announcements (no more scattered WhatsApp/email)
- **Threaded replies on announcements** — clarifying questions stay attached to the announcement instead of spinning up a side chat
- **Pinned/critical announcements** — deadline-critical posts don't get buried
- Role-based auth + route protection
- Supabase email/password login (accounts admin/advisor-created, not self-signup)

## Explicitly cut (would add monitoring work, not reduce it)

- Student activity heatmap
- Advisor performance/turnaround-time tracking
- Any per-advisor "response speed" leaderboard

## Parked for later (not core-purpose, revisit only if time allows)

- Team formation board ("looking for teammates")
- Skill tags on student profile for searchability
- Dark mode
- Email digest (weekly summary)

---

**Status:** Schema for all of the above is finalized and migrated (see `devsphere_schema.sql`). Currently starting Phase 2 — authentication.
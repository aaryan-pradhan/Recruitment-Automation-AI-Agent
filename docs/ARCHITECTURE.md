# Architecture

This document is a map of how the app is put together, for anyone reading the
code for the first time. See the main [README](../README.md) for what the app
does and how to run it locally.

## Layout

There is one Django app, `hiring_app`, inside the project `my_hiring_project`.

- `hiring_app/models.py` — persistence. `Campaign`, `Candidate`,
  `GoogleOAuthToken`, `LinkedInOAuthToken`.
- `hiring_app/services.py` — `HiringAutomator`, the class that talks to
  Gemini, Sheets, Gmail, and LinkedIn. Almost all business logic lives here.
- `hiring_app/views.py` — a thin controller layer over the two files above.
- `hiring_app/tasks.py` — Celery tasks for interview invites, outcome
  emails, and the daily resume-retention purge.

## Data model

- **`Campaign`** — one hiring campaign, keyed by a UUID (used in internal
  URLs) and a separate unguessable `public_token` (used in the public
  `/apply/<public_token>/` URL, so the public link never reveals the
  internal UUID). Holds the generated JD text and the AI-derived
  `scoring_keywords` every applicant is scored against.
- **`Candidate`** — one applicant to one `Campaign`, unique on
  `(campaign, email)` so a second application from the same address updates
  the existing row instead of creating a duplicate.
- **`GoogleOAuthToken` / `LinkedInOAuthToken`** — one row per user, holding
  encrypted OAuth credentials (Fernet, via `FIELD_ENCRYPTION_KEY`). Both
  integrations are entirely optional and scoped per-user — there is no
  shared fallback credential.

Ownership is enforced as a database constraint: every view resolves a
campaign via `get_object_or_404(Campaign, pk=campaign_id, owner=request.user)`,
so a campaign belonging to another user simply 404s.

## Request flow

1. **Draft** — recruiter enters a role + experience level; Gemini drafts a
   JD.
2. **Launch** — launching a campaign derives scoring keywords from the final
   JD (a second Gemini call), generates the public apply-page link, and
   optionally creates a tracking Sheet / cross-posts to LinkedIn.
3. **Apply** — candidates hit the public, unauthenticated `/apply/<token>/`
   page. Uploads are validated (PDF magic bytes, size cap) before anything
   else touches them, then parsed and scored against the campaign's
   keywords.
4. **Schedule** — recruiter selects candidates; `.ics` calendar invites go
   out as a background Celery task.
5. **Outcome** — recruiter sends offers/rejections in bulk (gated behind a
   typed confirmation, since it emails every candidate and can't be undone).

## Background tasks

Interview invites and outcome emails run as Celery tasks rather than inline
in the request, so a large batch doesn't hit gunicorn's request timeout.
Idempotency is enforced at the database level: each task re-queries for
candidates whose `invite_sent_at` / `outcome_sent_at` is still unset
immediately before running, so re-running the same task after a crash or a
redelivered broker message only processes whoever didn't get through last
time.

With no `REDIS_URL` set, Celery runs tasks eagerly (synchronously, in
process) — `manage.py runserver` alone is enough for normal local
development. A real worker is only needed to exercise the actual async path.

## Optional integrations

Google (Sheets export, Gmail send) and LinkedIn (JD cross-post) are both
strictly optional, connected per-user via OAuth, and never required for the
app to function — every step also works through Django's own configured
email backend and the self-hosted apply page when neither is connected.

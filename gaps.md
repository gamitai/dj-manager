# gaps.md: Skipped, Deferred & Unresolved Items

Log anything skipped, simplified, or left ambiguous. Keep entries short.

**Format:** `- [ ] <item> (Phase/Module) | Reason | Follow-up`
Tick `[x]` when resolved and note how.

## Skipped / Deferred

_None yet. Nothing has been built._

## Spec Gaps Found in the PRD (need a decision)

Found by comparing the PRD text against its own schema:

- [ ] `automation_logs` uniqueness: Section 3 says `UNIQUE(booking_id, trigger_type)`, the schema has `UNIQUE(booking_id, trigger_type, channel)`. Pick one.
- [ ] Repeating triggers: `meeting_reminder` is described as "24h and/or 2h", but one row per (booking, trigger) allows only one reminder. Needs distinct trigger types or a sequence number.
- [ ] Pre-event 7-day check-in has no `trigger_type` value in the schema.
- [ ] Meeting-only clients: `bookings.event_date` is `NOT NULL`, but a meeting can happen with only a tentative date. Also `meeting_confirmation` logs need a booking row.
- [ ] Per-client mute: PRD requires it, schema only has per-booking `automations_paused`. No per-client flag.
- [ ] Global automation on/off: no field in `automation_settings` for it.
- [ ] Signing token expiry (30 days): no `signing_token_expires_at` column. Invalidate-on-sign also needs a mechanism (rotate token or check `signed_at`).
- [ ] Signature storage: PRD says store raw PNG/vector, IP, user-agent, UTC time. `signature_audit` example has no `user_agent` or document hash field.
- [ ] Retry queue: Google Drive/Calendar retry is required but no table/queue is defined.
- [ ] Deposit due date: Module 4 says track due date, schema has no `deposit_due_at`.
- [ ] Balance reminder timing: Module 4 says 24h after event, Phase 5 uses `post_event_balance_reminder_days` (default 2). Pick one.
- [ ] 10-day review: "completed events" is undefined (signed only? not cancelled?). Confirm.
- [ ] Review links: schema is a generic JSONB list, but templates use a single `{{review_url}}`. Decide how multiple links map to one template.
- [ ] RLS: "scoped to `dj_id`" is stated, but no policies are written. The public signing page needs a token-based path that bypasses RLS safely.
- [ ] Pre-meeting reminder fields: office instructions (parking, floor code, arrival buffer) are Module 1 requirements, but `djs` only has address, map URL, and one note.
- [ ] Contract template fields: default package pricing and structured clauses (overtime rate, etc.) have no columns beyond `default_contract_terms`.
- [ ] Mobile choice: "React Native / Expo or PWA" is left open. Decide.
- [ ] WhatsApp provider: "Meta Cloud API or Twilio" is left open. Decide.
- [ ] Atomic post-sign pipeline: Phase 3 says "atomically", but Google/WhatsApp calls cannot be in one DB transaction. Needs a step-state model (idempotent steps + retries).

## External Prerequisites (not code)

- [ ] Meta WhatsApp Business account and template approval (templates need approval before use).
- [ ] Google Cloud OAuth consent screen / verification for Calendar and Drive scopes.
- [ ] Domain for `app.domain.com` signing links.
- [ ] Apple/Google developer accounts for TestFlight / internal track.

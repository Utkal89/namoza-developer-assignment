# Task 03 — Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

## Architecture

The form posts directly to a small serverless webhook endpoint (a single Cloud Function / Vercel
Edge Function), rather than using a native HubSpot embedded form or Zapier/Make. I'm choosing
**direct API call from a custom backend over native embed or Zapier/Make** for one reason: this flow
needs synchronous control over deduplication logic *before* the contact is written to HubSpot
(see below), and neither a native HubSpot form nor a no-code automation tool gives clean
conditional branching on a custom dedup key pre-write. Zapier/Make would work but adds 1-3
seconds of trigger-polling latency per step and a second point of failure outside our control —
unacceptable against a 2-minute WhatsApp SLA across three chained actions.

Flow: **Form submit → our webhook → HubSpot Contacts API (search-then-upsert) → Karix
WhatsApp API → Google Ads offline conversion / gtag.**

1. On submit, the front end fires the `consultation_form_submitted` dataLayer push immediately
   (for GTM/GA4), and in parallel POSTs `{name, phone, clinic_preference}` to our webhook.
2. The webhook first calls HubSpot's **Contacts Search API** filtering on the `phone` property
   (normalized to E.164 first) to check for an existing contact, rather than using HubSpot's
   default form-submission dedup, which is keyed on **email** — a field this form never
   collects. If a match exists, we `PATCH` that contact (updating Lead Status, appending a new
   Clinic Preference note, and timestamping the new enquiry) instead of creating a duplicate. If
   no match, we `POST` a new contact with Name, Phone, Clinic Preference, Source = "Google Ads
   - Consultation Landing Page", Lead Status = "New Enquiry".
3. Once HubSpot confirms (success response with contact ID), the webhook calls Karix's WhatsApp
   Business API to send the confirmation template message to the patient's number.
4. Once the WhatsApp send succeeds, the webhook fires a server-side Google Ads conversion
   (via Google Ads API / Enhanced Conversions for Leads, using the same phone number hashed)
   in addition to the client-side gtag fire, so the conversion isn't lost if the user closes
   the tab before any client-side pixel finishes firing.

Steps 2-4 run sequentially in the webhook (not fire-and-forget in parallel) so that if HubSpot
write fails, we don't send a WhatsApp confirmation referencing a record that doesn't exist.

## Phone deduplication — the trap

If two patients submit the same phone number with different names (e.g., a shared family phone,
common in Indian healthcare lead gen), HubSpot's **default** dedup (email-based) would silently
create two separate contact records with no link between them, fragmenting the patient's history
across the clinic group. Our webhook avoids this by never relying on HubSpot's native form
dedup at all — we do the phone-based search ourselves before any write. When a phone match
exists but the name differs, we don't overwrite the name; we append the new name and clinic
preference as a timestamped note/activity on the existing contact and flag it for manual review,
since auto-overwriting a name on a phone match risks merging two genuinely different people
who happen to share a household number.

## Biggest failure point & fallback

The biggest single point of failure is the **Karix WhatsApp API call** — third-party API uptime,
template approval issues, or the patient's number being unreachable on WhatsApp are all outside
our control, and a failure here shouldn't block the HubSpot write or the Ads conversion (which
already happened in steps 2-4 by that point). The fallback: wrap the Karix call in a retry (2
attempts, exponential backoff, ~20s total) and if both fail, write a `whatsapp_status: failed`
flag on the HubSpot contact and trigger a HubSpot workflow that creates a task for a front-desk
staff member to call the patient manually within the hour. This converts a silent failure into a
visible, owned task instead of a lost lead.

## Monitoring the 2-minute SLA

What could break it: Karix API latency/downtime, HubSpot API rate limiting on the search call,
cold-start latency on the serverless function, or template message approval delays from Meta. To
monitor: log a timestamp at form-submit and at WhatsApp-send-confirmed for every request, push
the delta to a lightweight dashboard (or just a logged metric in Datadog/CloudWatch), and alert
if delta exceeds 90 seconds (a buffer before the 2-minute SLA breaches) or if the failure-fallback
path triggers more than a handful of times per day, which would indicate a systemic issue with
Karix rather than one-off failures.

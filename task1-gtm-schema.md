# Task 01 — GTM Event Schema: OrthoNow

## 1. Full Event Schema

| Event Name | Trigger Type | Key Parameters | Feeds Into (GA4 / Audience) |
|---|---|---|---|
| `booking_step_complete` | Custom Event (dataLayer push, fired by front-end dev — see Section 2) | `step_number`, `step_name`, `clinic_location`, `specialty` (step 1 only), `preferred_date` (step 2 only) | Funnel Exploration report; Audience: "Started booking, did not finish" |
| `booking_confirmed` | Custom Event (dataLayer push on step 3 success, post API/CRM response) | `clinic_location`, `specialty`, `booking_id`, `value` (optional, fixed value for Ads bidding) | Conversions report; **imported into Google Ads** (see Section 3); Audience: "Converted patients" |
| `call_now_click` | Click — Just Links, trigger fires on Click Classes containing `call-now` or on `tel:` href prefix | `page_location`, `page_type` (home / clinic-page / landing-page), `clinic_location` (if on a clinic/location page) | GA4 Engagement report as a key event; Audience: "High-intent — called" |
| `whatsapp_chat_open` | Click — Just Links, trigger on Click URL contains `wa.me` | `page_location`, `clinic_location` (if determinable from page context), `widget_state` (opened) | Engagement report; Audience: "WhatsApp intent, no booking" |
| `patient_guide_download` | Custom Event, fired after the gating form (name+phone) submits successfully and the PDF download initiates | `form_location` (which page the gate sits on), `guide_title`, `lead_source` | Conversions report (secondary/micro-conversion); Audience: "Top-of-funnel lead — guide downloaders" |
| `clinic_page_view` | Page View (use GA4's automatic `page_view` enhanced with a custom parameter) OR a custom event if the 9 clinic pages don't share a clean URL pattern | `clinic_location`, `clinic_city`, `page_location` | Pages and screens report, filtered by `clinic_location`; Audience: "Researched specific clinic" |
| `blog_scroll_depth` | Scroll Depth trigger (built-in GTM trigger type), thresholds at 25/50/75/90% | `percent_scrolled`, `article_title`, `article_category` | Engagement report; Audience: "Engaged readers (75%+) — retargeting pool for content-to-booking nurture" |

Notes on trigger mechanics:
- `call_now_click` and `whatsapp_chat_open` are both genuinely native GTM capability — Click trigger types (Just Links / All Elements with a Click URL or Click Classes condition) can listen for these without any front-end code changes, as long as the buttons/links have consistent classes or hrefs.
- `clinic_page_view`, `blog_scroll_depth` are also native: Page View and the built-in Scroll Depth trigger don't require custom dataLayer code.
- `booking_step_complete`, `booking_confirmed`, and `patient_guide_download` are **not** native — they require the front-end developer to push structured events to `dataLayer` at the moment those interactions happen. GTM has no way to "see" a form step transition inside a JS-driven multi-step form (no full page load occurs between steps), so it cannot infer step completion on its own.

---

## 2. Booking Form Funnel Drop-off Tracking

### Why this can't be done with GTM triggers alone
The 3-step form almost certainly doesn't reload the page between steps (it's a single-page JS-driven flow showing/hiding step panels, or an SPA-style form). GTM's DOM-based triggers (Click, Form Submission, Element Visibility) can catch *some* of this — e.g. a "Next" button click — but they can't reliably capture "the user successfully completed and validated step 1" vs. "the user clicked Next but validation failed." That distinction requires the form's own JS validation logic to decide success, then explicitly notify the dataLayer. This is a front-end dev task, not a GTM-config task.

### What fires at each step

**Step 1 — Location + Specialty selected**
Trigger: a custom GTM trigger listening for a Custom Event named `booking_step_complete` where `step_number` equals `1`.
Front-end dev pushes this the moment the user has selected both a clinic and specialty and clicks "Continue" (after client-side validation passes):

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar",
  "specialty": "Knee & Joint Replacement"
}
```

**Step 2 — Contact details entered**
Same event name, `step_number: 2`, fired after the name/phone/preferred-date fields pass validation and the user clicks "Continue":

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Indiranagar",
  "specialty": "Knee & Joint Replacement",
  "preferred_date": "2026-07-04"
}
```

**Step 3 — Booking confirmed**
A *separate* event name (`booking_confirmed`, not another `booking_step_complete`), fired only after the backend/CRM call confirming the booking succeeds — not just on button click, so we don't count failed submissions as conversions:

```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar",
  "specialty": "Knee & Joint Replacement",
  "booking_id": "ON-48213"
}
```

### GTM configuration
- One Custom Event trigger on `booking_step_complete`, fanning out to a single GA4 Event tag that maps `step_number`, `step_name`, `clinic_location`, `specialty`, `preferred_date` as event parameters (all marked as user-scoped/event-scoped custom dimensions in GA4 Admin so they're queryable).
- A second Custom Event trigger on `booking_confirmed` → GA4 Event tag, with `booking_confirmed` marked as a **Key Event** in GA4.

### Surfacing drop-off in GA4 Funnel Exploration
Build a funnel with four open steps:
1. `booking_step_complete` where `step_number = 1`
2. `booking_step_complete` where `step_number = 2`
3. `booking_confirmed`

Set the funnel to "Open funnel" (not closed) so GA4 doesn't require strict step-1-before-step-2 ordering issues to break the count, and enable "Show elapsed time" to see how long users sit on step 2 (the contact-details step, typically the biggest drop-off point in healthcare forms because it asks for a phone number). The funnel visualization will show the % drop-off between each step directly, and you can break it down by `clinic_location` or by traffic source (e.g., is the landing page driving worse step-2 completion than the homepage form?) using a secondary dimension.

---

## 3. Conversion Action to Import into Google Ads

**Import `booking_confirmed`, not `call_now_click`, `patient_guide_download`, or `booking_step_complete`.**

Reasoning:
- `booking_confirmed` is the only event that represents a fully realized, validated business outcome (a confirmed appointment) — it's the cleanest signal for Smart Bidding to optimize toward without rewarding noisy or low-intent behavior.
- `call_now_click` is tempting because phone calls are high-intent in healthcare, but it fires on *click*, not on an actual answered/qualified call, and would need call tracking (e.g., a CallRail-style number swap) to be a trustworthy conversion signal. Importing it now would let Smart Bidding optimize toward people who click-to-call but never actually connect or book.
- `patient_guide_download` is a useful **secondary/micro-conversion** for retargeting and content scoring, but it's top-of-funnel — optimizing ad spend toward "downloaded a PDF" would likely lower lead quality, not improve it.
- `booking_step_complete` (steps 1–2) are good for diagnosing funnel friction internally, but importing a mid-funnel, non-committal step as a conversion action would tell Google Ads to chase form-starters, not form-finishers, which historically inflates cost-per-lead without improving actual booked consultations.

Once volume on `booking_confirmed` is healthy (generally 30+ conversions/month per campaign for Smart Bidding to have enough signal), this becomes the primary conversion action; `patient_guide_download` can be imported later as a secondary action for upper-funnel campaigns once there's a need to scale reach.

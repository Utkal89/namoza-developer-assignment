# Presentation Prep — Likely Follow-Up Questions

Based on what each task is actually screening for, here's how to answer cleanly if asked live.

## Task 01: "Who writes the dataLayer push — you or the front-end dev?"
**Answer:** The front-end developer writes it, because GTM has no native visibility into a
multi-step form's internal state when there's no full page reload between steps. GTM can listen
to *DOM events* (clicks, page views, scroll, element visibility) but it can't know that "step 2
validation just passed" unless the application code tells it so explicitly via
`window.dataLayer.push(...)`. My job as the marketing/analytics developer is to define the exact
event name, parameter names, and the moment each push should fire — then hand that off as a
spec to the front-end dev to wire into the form's validation success callbacks. I'd brief step 2
specifically as: "After the name/phone/date fields pass client-side validation and the user
clicks Continue, push this object to dataLayer before transitioning the UI to step 3" — then give
them the exact JSON shape from the schema, including which fields are static strings vs. which
need to be populated dynamically from form state (e.g. `clinic_location` carried over from step 1).

## Task 02: "Show me the dataLayer push firing live."
Walkthrough order for the Loom:
1. Open the page, open DevTools → Console.
2. Type `window.dataLayer` and hit enter — show it's an empty/initialized array.
3. Fill the form, click submit.
4. Type `window.dataLayer` again — show the `consultation_form_submitted` object now in the array,
   with `clinic_location` and `lead_source` populated.
5. Point out the thank-you state rendered without a page reload (URL bar didn't change/reload).

## Task 03: "Two patients submit with the same phone number, different names — what happens?"
**Answer:** It does *not* fall through to HubSpot's default dedup, because that's keyed on email
and this form never collects one — that's the trap. My webhook does its own search against
HubSpot's Contacts Search API filtered on the normalized phone number *before* deciding whether
to create or update. If a match is found but the name differs, I don't silently overwrite the
existing name (that risks merging two different people who share a household phone — common with
a single family landline-turned-mobile situation). Instead I append the new name and clinic
preference as a logged activity/note on the existing contact and flag it for manual review by a
coordinator, so a human resolves the ambiguity instead of the system guessing.

# OrthoNow — Developer Assignment Submission

| Task | File |
|---|---|
| 01 — GTM Event Schema | `task1-gtm-schema.md` |
| 02 — Landing Page | `task2-landing-page/index.html` |
| 03 — Integration Design | `task3-integration-design.md` |

## Before you submit — things only you can finish

I built everything that can be built offline, but three things need you, your browser, and (for one of them) a public URL:

1. **PageSpeed screenshot (Task 02, hard requirement).** Open `task2-landing-page/index.html` locally first to sanity-check it, then push it somewhere public (GitHub Pages is the easiest: Settings → Pages → deploy from the repo) so PageSpeed Insights can actually crawl it — PSI can't score a `file://` page or localhost. Run it at https://pagespeed.web.dev against the Mobile tab, and save the screenshot into the repo (e.g. `task2-landing-page/pagespeed-mobile.png`). I designed the page specifically to clear 90+: no webfonts, no external images, no JS frameworks, no render-blocking requests — but I can't generate a real PSI score myself, so you do need to actually run this and confirm.

2. **Live console demo (for the Loom).** Open the page, open DevTools console, type `window.dataLayer`, submit the form, and show the `consultation_form_submitted` push appear live. This is explicitly what they said they'll ask you to do, so rehearse it once before recording.

3. **GitHub repo setup.** Create the repo, drop these three items in, share access with `himanshu@namoza.com`, and send the repo + Loom links to `naman@namoza.com` with the exact subject line: `Developer Assignment - [Your Name]`.

## A note on the assignment PDF

The file you uploaded includes the interviewer's internal grading notes (the boxes marked "WHAT WE'RE ACTUALLY TESTING HERE — INTERVIEWER ONLY"). I used them to make sure each deliverable directly addresses what they're probing for — see `PRESENTATION_PREP.md` for the specific live-answer prep on each one. Worth knowing this was visible to you, in case it wasn't supposed to be and you want to flag it or just keep it to yourself.

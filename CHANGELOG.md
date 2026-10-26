# Changelog

## v1.1 (2026-10-27)

Fixes from an independent review of v1. Every fix has a test; the suite was rerun from a clean start.

### Review requests (the GoHighLevel recipe)

- **A customer could get two requests and two reminders.** HighLevel lets a contact into a workflow with an appointment trigger again for every appointment, even while they are still in it, and a won job and a showed appointment often land on the same day. The guard only looked for `review-requested`, which is added after the 2 hour wait, so both starts passed it. Workflow 1 now adds `review-queued` right after the guard, before the wait, and the guard looks for either tag. The test starts workflow 1 twice for the same customer.
- **Stop on Response is now off.** HighLevel ends the workflow when the contact answers "a message sent from this specific workflow", but it does not say what happens to a message sent during the 2 hour wait, before the request. If such a message ended the workflow, a customer who wrote in right after the job (often to complain) would never get the request. The "Replied or clicked?" step already stops the reminder for anyone who replied, so the setting is not needed. The test harness now models HighLevel's definition, and a customer who writes during the wait still gets the request.
- The recipe now says how to put the trigger link in a message (the copy icon next to its Link Key, which copies a `{{trigger_link.<id>}}` merge code), marks `{{message.body}}` as documented for Customer Replied, and updates what could not be confirmed. The advice to set the built-in Send Review Request action's Maximum retries to 0 was dropped (the help center does not show that 0 is allowed), and the claim that its repeats do not stop on a reply now says the help center does not mention replies.

### Replies (n8n)

- **A message that may ask us to stop could still be followed by a text.** "Please stop texting me about this" from a customer a person already had (or who had reached the reply limit) left a reply that was waiting for the morning in place, and within 6 hours of an earlier Slack note it was dropped without any alert. Such a message now cancels the waiting reply and always reaches Slack, with a line to set DND by hand.
- **An email opt out could switch off the wrong channel.** The Custom Webhook payload has no channel, so n8n reads it from the matching message in the conversation. When that failed (history not loaded, or the email body not matching character for character), a contact with a phone number was treated as an SMS reply, and an email UNSUBSCRIBE set SMS DND. An opt out on a channel n8n cannot tell now sets DND for SMS and email, and the matching ignores whitespace and HTML tags.
- **A refused Slack alert was silent.** The Slack node continued on error and the run ended as a success, so a complaint alert could vanish. The run now fails with the whole alert in the error.
- **An empty `SLACK_WEBHOOK_URL` was allowed for replies.** Complaints, questions and hand offs then reached nobody while the customer was told someone would get back to them. It is now a required setting.
- **A reply without text was dropped without a word,** and an unfilled `{{message.body}}` would have been sent to Claude as the reply. Both now fail the run at "Stop: No Reply Text" with what to do.
- **The reply guard missed ways to steer a review.** "Give us a chance to fix it before you post anything online", "send your feedback to us" and "10% off" passed it. It now also blocks posting, online, public, Google, Yelp, Facebook, feedback, surveys, hold off, vouchers, reimbursement, waived or complimentary work, "on us" and percentages.
- **The customer's text could close the tag it sits in.** A reply containing `</customer_reply>` could put words outside the data tags of the Claude request. Those tags are now taken out of customer text and names.
- **Quiet hours were decided before Claude answered.** With retries, a reply decided at 7:59 PM could go out after 8 PM, or be scheduled for a morning time that had already passed. The window is now checked again after Claude answers.
- The mock now also answers the message history in the flat shape the OpenAPI file documents, not only the nested shape seen from the live API (the workflow reads both), and Slack can refuse a post the way a removed webhook does (404 `no_service`).

### Docs

- The Business Profile help link pointed at a redirect, and the Endorsement Guides link at the wrong eCFR subchapter (Part 255 is in subchapter B). Both fixed; the citations were checked against the live pages again.
- "Retried 3 times" now reads "tried up to 3 times": n8n's 3 tries include the first one.

## v1 (2026-10-26)

First build: review requests and service recovery for GoHighLevel. The GoHighLevel side is a recipe for three native workflows; the n8n side handles replies with Claude, the owner alerts, the recovery tasks and a weekly Slack digest. Tested end to end against mock APIs, with the native workflows played from the recipe.

### GoHighLevel (native workflows, recipe in ghl/native-workflows.md)

- Workflow 1 starts on Opportunity Status Changed (Moved To Status: Won) or Appointment Status (Showed), skips anyone already asked, waits 2 hours inside business hours, tags `review-requested`, texts the review link (email when the contact has no phone), tells n8n, waits 3 days and sends one reminder unless the contact replied or clicked. Stop on Response is on.
- Workflow 2 tags every reply `review-replied` and forwards it to n8n with the shared secret in the `X-Review-Secret` header (Custom Webhook, Key/Value body).
- Workflow 3 tags a click on the review trigger link.
- Every trigger, filter, setting and action name was checked against help.gohighlevel.com. What could not be confirmed is listed at the end of the recipe.
- The recipe ends with a machine-readable copy that the tests read, so the page and the tests cannot drift apart.

### n8n

- One events webhook for `request_sent`, `reminder_sent` and `customer_replied`, checked with the shared secret (header or `customData.secret`). Both the Custom Webhook payload and the standard Webhook payload are read, as JSON or as a form body.
- Request and reminder events save the send time in two text custom fields on the contact, so the digest counts from GoHighLevel itself.
- Replies: repeat deliveries skipped, contacts outside the flow and replies after `REPLY_WINDOW_DAYS` ignored, STOP words handled in code before any AI (at any time), a Slack note for customers a person already has, a hand off after `MAX_AI_REPLIES`.
- Claude (`claude-sonnet-5` by default) sorts each reply into praise, complaint, question, opt out or other through a forced, strict tool call. Its reply text is checked in code and replaced by a fixed text if it has a link, mentions reviews, ratings or stars, asks to change or remove anything, or offers something of value.
- Complaints get a GoHighLevel task for the owner with a due date, a note, two tags, a Slack alert with the issue in one line and an apology that a person will call. The review request is never touched.
- SMS replies outside the quiet hours window are scheduled in GoHighLevel (`scheduledTimestamp`) and cancelled when the customer opts out or writes again before the morning.
- Weekly Slack digest (Mondays at 8 AM, or on demand): requests, reminders, replies by type, complaints and open recovery tasks.
- Every failure ends the run as failed with the reason, for the error alert workflow.

### Not built

- Claude-drafted replies to new Google reviews. The GoHighLevel API v2 has no documented way to read reviews (see the README).

### Found and fixed while testing

- **STOP was ignored after the reply window.** A STOP that arrived more than `REPLY_WINDOW_DAYS` after the request ended at the window check and set no DND. The STOP check now runs first, and the test sends STOP to a customer asked 40 days earlier.
- **A customer already with a person flooded Slack.** Every message in a live conversation raised its own Slack note. Now it is at most one note per customer every 6 hours, and the test checks a second message stays quiet.
- **The quiet hours line mixed time zones.** "They will get this at 8:00 AM" sat next to the owner's task due time, which is in the business time zone. It now says "8:00 AM their time".
- **The digest could run twice.** The task loads run once per recovery contact. With an error output, a partial failure would put items on both outputs and run Build Digest twice. Failures now stay on the main output with the error, and Build Digest reports them once.
- **Paginated search errors were unreadable.** n8n's error item for a paginated request has no `message`, so the alert showed a raw JSON blob. The alert now reads "500 - " plus the response.
- **The task loads went out in parallel.** That burst refused connections on the test machine, and it is the kind of burst GoHighLevel rate limits. They now go one at a time, 200 ms apart. The test mock now keeps connections open and accepts a longer queue of waiting connections; before that, a retry now and then stalled until n8n's timeout while other containers were busy, and the checks now count every retry per call so a lost one shows up.
- **Claude saw a reply the customer never got.** When a customer wrote again during quiet hours, the conversation sent to Claude included the thank-you that was still waiting for the morning (and was about to be cancelled), as if the customer had read it. Messages with the status `scheduled` are now left out, and the test checks it.
- **The DND update used a computed key inside an n8n expression.** It is now built in the Code node, where it is plain JavaScript.
- **Two message templates had a double period** ("Summit Roofing Co..").

### Proving the checks

The suite was also run once against a deliberately broken copy: the reply text check in n8n switched off, and the reminder rule in the recipe's machine-readable copy changed to read the `review-reply-complaint` tag (review gating). 73 of 78 checks passed: the recipe's gating rule check, the reminder rule check, both reply guard scenarios and the global "no message touched the review" check failed, as they should. One check passed that should not have: the page still showed the old rule in its prose. That check now compares every step and tag condition in the prose with the copy the tests play.

# GoHighLevel native workflows: build recipe

The GoHighLevel side of this build runs as three native workflows, built by hand in **Automation > Workflows**. They send the review request and the single reminder, and they tell n8n what happened. n8n handles the replies (Claude, Slack, tasks) and the weekly digest. This page is the exact recipe.

Names of triggers, filters, actions and settings were checked against HighLevel's help center (help.gohighlevel.com) on October 26, 2026, and the riskiest ones again on October 27, 2026. Anything that could not be confirmed there is marked *(not confirmed)* and listed in [What could not be confirmed](#what-could-not-be-confirmed). The tests read the machine-readable copy at the end of this page.

## The rule this recipe protects

Every customer gets the same review request with the same link, whatever they think of the job. Google's policy does not allow businesses to "Discourage or prohibit negative reviews, or selectively solicit positive reviews from customers", and the FTC says that asking only happy customers ("review gating") can violate the FTC Act (see the README for the sources). So:

- Nothing in these workflows looks at what the customer thinks. No rating question, no "were you happy?" step, no branch on a reply type.
- The request text only asks for a review. It does not ask unhappy customers to contact you first, because that steers bad experiences away from Google.
- No discount, gift or entry into a draw for a review.
- The reminder stops for anyone who replied (any reply, happy or not) or clicked the link. That is the same rule for everyone.
- HighLevel has no "review gating" setting to switch off. Do not add a rating pre-screen page or a feedback form in front of the Google link, even though some HighLevel guides suggest routing low ratings to an internal form.

n8n never changes any of this. It reads the tags these workflows add but never adds or removes them, and it never sends, repeats or withdraws the review link.

## Before you start

1. **Custom value for the review link.** **Settings > Custom Values > New Custom Value**. Name `Google Review Link`, value: your Google review link (in your Google Business Profile, "Ask for reviews" gives you the link). The key becomes `{{custom_values.google_review_link}}`.
2. **Trigger link.** **Marketing > Trigger Links > Links > Add Link** (older accounts: **Conversations > Trigger Links**). Name `Google review`. For Link URL, click the `{}` button and pick the Google Review Link custom value. A trigger link records each contact's click and fires the Trigger Link Clicked trigger. A raw URL or the custom value pasted into a message does not, so always put the trigger link itself in the message: in the Trigger Links list, click the copy icon next to its **Link Key**. That copies a merge code such as `{{trigger_link.AbC123xyz}}` (the ID is your link's).
3. **Two custom fields** for n8n to write the send times into. **Settings > Custom Fields**, object Contact, **Create field**, type single line text (called Short Text in some accounts *(not confirmed which label your account shows)*): `Review Request Sent` and `Review Reminder Sent`. Text, not a date picker, because the API documents text values for custom fields. [docs/setup.md](../docs/setup.md) shows how to get their IDs for n8n.
4. **The n8n webhook.** Import and activate the n8n workflow first. Its URL is `https://YOUR-N8N-DOMAIN/webhook/ghl-review-events`, and the shared secret is the value of `REVIEW_WEBHOOK_SECRET` in n8n.
5. **Premium actions.** Custom Webhook is a premium workflow action: each sub-account gets 100 free executions in total, then it is $0.01 per execution (Workflow Pro plans are cheaper at volume). The agency must turn premium actions on under **Settings > Company > Workflow - Premium Features**. Each customer costs about 2 to 4 executions (request, maybe a reminder, replies).

Tags are created on the fly by the Add Contact Tag action. If you change the tag names, set `TAG_PREFIX` in n8n to match (the default prefix is `review`).

## Review Request 1: Ask after the job

Asks every customer for a Google review once the job is won or the appointment showed, then reminds them once.

### Triggers

Add each with **+ Add New Trigger**, then **+ Add Filters**, then **Save Trigger**. One workflow can have several triggers.

| Trigger | Filters |
|---|---|
| Opportunity Status Changed | In Pipeline: your jobs pipeline (for example "Roofing Jobs"); Moved To Status: Won |
| Appointment Status | In Calendar Filter: your job or inspection calendar; Filter By Appointment Status: Showed |

### Settings tab

| Setting | Value |
|---|---|
| Allow Re-entry | Off |
| Stop on Response | Off (the "Replied or clicked?" step stops the reminder for anyone who replied, so no setting lets a message from the customer, sent before or after the request, decide whether they get the request) |
| Timezone | Contact Timezone |
| Time Window | Specific Time on, Start Time 9:00 AM, End Time 6:00 PM, Include Days Monday to Saturday |

### Actions, in order

1. **If Else**, named "Already asked?". Condition: Contact Tags includes any of `review-requested`, `review-queued`. *(The exact condition label is not confirmed.)*
   - Branch "Yes": **Remove from Workflow**, option "Current Workflow".
   - Branch "None": carry on with the next step.
2. **Add Contact Tag**, named "Tag: queued". Tag: `review-queued` (type it and click "Add New Tag" the first time).
3. **Wait**, named "Wait 2 hours". Choose "A set period of time", 2 hours. Turn on **Advance Window**: Resume On: Mon, Tue, Wed, Thu, Fri, Sat, Resume Between Hours: 9:00 AM to 6:00 PM.
4. **Add Contact Tag**, named "Tag: requested". Tag: `review-requested` (type it and click "Add New Tag" the first time).
5. **If Else**, named "Has a phone number?". Condition: Contact Details: Phone Is not empty. *(The Contact Details group and the Is not empty operator are confirmed; the exact field label is not.)*
   - Branch "Yes" (text):
      1. **Send SMS**, named "Review request (SMS)". Message: the "request_sms" text below.
      2. **Custom Webhook**, named "Tell n8n: request sent". Settings in "Custom Webhook settings" below, payload "request_sent".
      3. **Wait**, named "Wait 3 days". Choose "A set period of time", 3 days. Turn on **Advance Window**: Resume On: Mon, Tue, Wed, Thu, Fri, Sat, Resume Between Hours: 9:00 AM to 6:00 PM.
      4. **If Else**, named "Replied or clicked?". Condition: Contact Tags includes any of `review-replied`, `review-link-clicked`. *(The exact condition label is not confirmed.)*
         - Branch "Yes": **Remove from Workflow**, option "Current Workflow".
         - Branch "None": carry on with the next step.
      5. **Add Contact Tag**, named "Tag: reminded". Tag: `review-reminded` (type it and click "Add New Tag" the first time).
      6. **Send SMS**, named "Review reminder (SMS)". Message: the "reminder_sms" text below.
      7. **Custom Webhook**, named "Tell n8n: reminder sent". Settings in "Custom Webhook settings" below, payload "reminder_sent".
   - Branch "None" (email, for contacts without a phone number):
      1. **Send Email**, named "Review request (email)". Message: the "request_email" text below.
      2. **Custom Webhook**, named "Tell n8n: request sent". Settings in "Custom Webhook settings" below, payload "request_sent".
      3. **Wait**, named "Wait 3 days". Choose "A set period of time", 3 days. Turn on **Advance Window**: Resume On: Mon, Tue, Wed, Thu, Fri, Sat, Resume Between Hours: 9:00 AM to 6:00 PM.
      4. **If Else**, named "Replied or clicked?". Condition: Contact Tags includes any of `review-replied`, `review-link-clicked`. *(The exact condition label is not confirmed.)*
         - Branch "Yes": **Remove from Workflow**, option "Current Workflow".
         - Branch "None": carry on with the next step.
      5. **Add Contact Tag**, named "Tag: reminded". Tag: `review-reminded` (type it and click "Add New Tag" the first time).
      6. **Send Email**, named "Review reminder (email)". Message: the "reminder_email" text below.
      7. **Custom Webhook**, named "Tell n8n: reminder sent". Settings in "Custom Webhook settings" below, payload "reminder_sent".

HighLevel lets a contact into a workflow with an appointment trigger once per appointment, even while they are still in it, whatever Allow Re-entry says, and a won job and a showed appointment often land on the same day. The first two steps stop a second request: the guard looks for either tag, and `review-queued` is added at once, before the 2 hour wait, so a second start for the same customer stops at the guard. Remove the `review-requested` and `review-queued` tags by hand if you want to ask a customer again after a new job.

The branches do not join again, so the reminder steps appear once in each branch. Build both.

## Review Request 2: Forward replies

Tags every reply from a contact who got the request and sends it to n8n. The tag is added here, for every reply, before n8n reads it.

### Triggers

Add each with **+ Add New Trigger**, then **+ Add Filters**, then **Save Trigger**. One workflow can have several triggers.

| Trigger | Filters |
|---|---|
| Customer Replied | Has Tag: review-requested; Doesn't Have Tag: review-opted-out; Reply Channel: SMS |
| Customer Replied | Has Tag: review-requested; Doesn't Have Tag: review-opted-out; Reply Channel: Email |

### Settings tab

| Setting | Value |
|---|---|
| Allow Re-entry | On (every reply must reach n8n) |
| Stop on Response | Off |

### Actions, in order

1. **Add Contact Tag**, named "Tag: replied". Tag: `review-replied` (type it and click "Add New Tag" the first time).
2. **Custom Webhook**, named "Send the reply to n8n". Settings in "Custom Webhook settings" below, payload "customer_replied".

## Review Request 3: Link clicked

Marks contacts who clicked the review link, so they get no reminder.

### Triggers

Add each with **+ Add New Trigger**, then **+ Add Filters**, then **Save Trigger**. One workflow can have several triggers.

| Trigger | Filters |
|---|---|
| Trigger Link Clicked | Trigger link: Google review |

### Settings tab

| Setting | Value |
|---|---|
| Allow Re-entry | Off |

### Actions, in order

1. **Add Contact Tag**, named "Tag: link clicked". Tag: `review-link-clicked` (type it and click "Add New Tag" the first time).

## Messages

Paste these into the Send SMS and Send Email actions. Where it says `[trigger link: Google review]`, paste the "Google review" trigger link's merge code (the `{{trigger_link.<id>}}` code from the copy icon next to its Link Key, see "Before you start"). Change the business name and wording to yours, but keep them neutral: ask for "your experience", never for a good or 5-star review.

**request_sms** (SMS)

```text
Hi {{contact.first_name}}, thank you for choosing Summit Roofing Co. If you have a minute, would you share your experience in a Google review? [trigger link: Google review] Reply STOP to opt out.
```

**reminder_sms** (SMS)

```text
Hi {{contact.first_name}}, a quick reminder from Summit Roofing Co.: if you have a minute, a Google review of your experience helps other homeowners decide. [trigger link: Google review] Reply STOP to opt out.
```

**request_email** (Email)

Subject: `A quick favor from Summit Roofing Co.`

```text
Hi {{contact.first_name}},

Thank you for choosing Summit Roofing Co. If you have a minute, would you share your experience in a Google review? It helps other homeowners decide.

[trigger link: Google review]

Thank you,
The Summit Roofing Co. team
```

**reminder_email** (Email)

Subject: `A quick reminder from Summit Roofing Co.`

```text
Hi {{contact.first_name}},

A quick reminder: if you have a minute, a Google review of your experience with Summit Roofing Co. helps other homeowners decide.

[trigger link: Google review]

Thank you,
The Summit Roofing Co. team
```

## Custom Webhook settings

Every **Custom Webhook** action in these workflows uses the same settings. Only the payload differs.

| Setting | Value |
|---|---|
| Event | `POST` (HighLevel calls this body a "form-style payload" of Key/Value pairs and does not say which content type it sends; n8n reads JSON and form bodies alike. Do not switch to CUSTOM, which needs a raw body) |
| URL | `https://YOUR-N8N-DOMAIN/webhook/ghl-review-events` |
| Authorization | None |
| Headers | Key `X-Review-Secret`, value: your `REVIEW_WEBHOOK_SECRET` |
| Body | Key/Value pairs, one row per field of the payload, values as below (use the merge field picker for the `{{...}}` values) |

Key/Value rows let HighLevel build the body itself, so a customer's quote marks or line breaks cannot break it. Avoid a raw JSON body with `{{message.body}}` in it.

**request_sent**

| Key | Value |
|---|---|
| `event` | `request_sent` |
| `locationId` | `{{location.id}}` |
| `contactId` | `{{contact.id}}` |

n8n receives:

```json
{"event": "request_sent", "locationId": "{{location.id}}", "contactId": "{{contact.id}}"}
```

**reminder_sent**

| Key | Value |
|---|---|
| `event` | `reminder_sent` |
| `locationId` | `{{location.id}}` |
| `contactId` | `{{contact.id}}` |

n8n receives:

```json
{"event": "reminder_sent", "locationId": "{{location.id}}", "contactId": "{{contact.id}}"}
```

**customer_replied**

| Key | Value |
|---|---|
| `event` | `customer_replied` |
| `locationId` | `{{location.id}}` |
| `contactId` | `{{contact.id}}` |
| `body` | `{{message.body}}` (HighLevel's help center describes it as "the content of the reply that triggered the workflow" for a Customer Replied workflow on SMS) |

n8n receives:

```json
{"event": "customer_replied", "locationId": "{{location.id}}", "contactId": "{{contact.id}}", "body": "{{message.body}}"}
```

**No premium actions?** The standard **Webhook** action is free but has no headers setting. Point it at the same URL and add two Custom Data items: `secret` with your `REVIEW_WEBHOOK_SECRET`, and `event` with `request_sent`, `reminder_sent` or `customer_replied`. n8n reads that payload too: `location.id` and `message.body` are in HighLevel's sample payload, while `contact_id` and the `customData` object that holds your items are *not confirmed* by the help center (check one call with a request bin first). Treat the secret like a password either way.

## Tags

| Tag | Added by | What it means |
|---|---|---|
| `review-queued` | Workflow 1 | Started workflow 1. A second start stops at the guard |
| `review-requested` | Workflow 1 | Got the review request. Never asked again |
| `review-reminded` | Workflow 1 | Got the one reminder |
| `review-replied` | Workflow 2 | Replied at least once (any reply). No reminder |
| `review-link-clicked` | Workflow 3 | Clicked the review link. No reminder |
| `review-reply-praise`, `review-reply-complaint`, `review-reply-question`, `review-reply-other` | n8n | What Claude read in a reply |
| `review-needs-recovery` | n8n | Complained. The owner has a task to call them |
| `review-needs-human` | n8n | A person should answer. The assistant stays quiet |
| `review-opted-out` | n8n | Asked to stop. DND is on for that channel |

Workflow 1, which sends the request and the reminder, only reads tags from the first five rows. n8n only adds the others. That split is what keeps the reply side from ever changing who gets the request or the reminder. (Workflow 2 skips contacts tagged `review-opted-out`, so their later messages do not reach n8n; that does not change who gets the request or the reminder.)

## Why not the built-in Send Review Request action

HighLevel has its own **Send Review Request** action (Review Type: E-mail or SMS), with the text, the review link and the repeat schedule set under **Reputation > Settings** ("Until clicked, repeat this every", "Maximum retries"). It is sentiment blind too, so it is a valid choice. This recipe uses Send SMS and Send Email with a trigger link instead, because:

- its repeats are documented to stop when the customer clicks; the help center does not say they stop on a reply, so a customer who replied with a complaint may still get them;
- the email fallback for contacts without a phone is documented only for the manual multi-channel send, not for the workflow action;
- n8n needs a "request sent" and "reminder sent" event for the weekly digest, which is easiest when each send is its own step.

If you use it anyway, drop the reminder steps here and set Maximum retries to 1, so its one repeat is the only reminder, and accept that it stops on a click but maybe not on a reply.

## Test it

1. For the test, shorten both Wait steps to 1 minute (put them back afterwards). Create a test contact with your own phone number, put it in your jobs pipeline and mark the opportunity Won.
2. A minute later the contact gets the text and the `review-requested` tag. The n8n execution list shows a `request_sent` run, and the contact's Review Request Sent field is filled.
3. Reply "Thanks, great job" from your phone. Workflow 2 adds `review-replied`, n8n answers with a short thank-you and tags `review-reply-praise`. After the second wait, workflow 1 sees the `review-replied` tag and ends for that contact, so there is no reminder.
4. With a second test contact, reply with a complaint and check the Slack alert, the task and the note.
5. With a third, do not reply, and check that exactly one reminder arrives after the second wait, inside business hours, and no second one after it.
6. Mark a showed appointment for the first test contact as well, and check that no second request goes out (the guard sees `review-queued`).

## What could not be confirmed

- The exact text of a trigger link's merge code. The help center says the copy icon next to the Link Key "place[s] the merge code on your clipboard" to paste into SMS, emails or workflow actions, but shows no example; the `{{trigger_link.<id>}}` form comes from HighLevel's API docs (`links.json` on GitHub). This recipe writes `[trigger link: Google review]` where you paste it.
- That `{{message.body}}` also carries the text of an email reply. The help center says it "provides the content of the reply that triggered the workflow" in a Customer Replied workflow filtered to SMS; email is not mentioned. If an email reply arrives in n8n without text, the run fails at "Stop: No Reply Text" and says so.
- Which content type the Custom Webhook action sends for a POST Key/Value body (HighLevel only says "form-style payload"). The n8n webhook reads JSON and form bodies, and the tests send both.
- How many times HighLevel retries a Custom Webhook that fails ("exponential backoffs", no count given). n8n answers at once and drops repeat deliveries, so a retry does no harm.
- The exact If Else condition labels for "Contact Tags includes" and for "Phone Is not empty". The If Else article confirms the "None" branch; "Includes" and "Is not empty" come from an older help page that is now only in the Web Archive.
- The full list of Reply Channel options on Customer Replied. SMS and Email are named in the help articles.
- Whether a contact can be in workflow 1 twice when its Opportunity Status Changed trigger and its Appointment Status trigger fire close together. The help center says appointment triggers let a contact in "multiple times even if the contact is already present in the workflow", so the recipe guards against it with `review-queued` either way.
- The standard Webhook action's `contact_id` key and `customData` object (see "No premium actions?" above).
- Whether `{{reputation.review_link}}` works in a normal Send SMS action. The help center documents it only for the request templates under Reputation > Settings, and a feature request for such a merge field is still open. This recipe uses its own custom value instead.

Confirmed on help.gohighlevel.com: the triggers Opportunity Status Changed (filters In Pipeline, Moved To Status), Appointment Status (filters In Calendar Filter, Filter By Appointment Status, status Showed), Customer Replied (filters Has Tag, Doesn't Have Tag, Reply Channel) and Trigger Link Clicked (filter Trigger link); several triggers per workflow; the Settings tab options Allow Re-entry (with the exception that appointment and invoice triggers always let a contact in once per appointment or invoice), Stop on Response (it ends the workflow when the contact responds "to a message sent from this specific workflow"), Timezone and Time Window (it holds back messages, not internal actions such as adding a tag); the actions Add Contact Tag, Send SMS, Send Email, If Else, Wait ("A set period of time" with Advance Window, Resume On, Resume Between Hours), Remove from Workflow (option Current Workflow), Webhook and Custom Webhook (Event, Method, Authorization, Headers, Key/Value body); custom values (`{{custom_values.<name>}}`), trigger links and the copy icon next to the Link Key, the `{{message.body}}` merge field, STOP and the other standard opt out keywords turning DND on (after which queued texts fail), the Send Review Request action and the Reputation settings; and Custom Webhook pricing (100 free premium executions in total, then $0.01 each). HighLevel also has a contactless "New Review Received" trigger (filters Review SPAM, Review Source, Review Rating); see the README for why replies to reviews are not part of this build.

## Machine-readable copy (used by the tests)

```json
{
  "note": "Machine-readable copy of this recipe. tests/e2e_mock_test.py reads it, plays these workflows against the mock GoHighLevel API and sends the same webhook calls to n8n. It cannot be imported into GoHighLevel.",
  "reviewLink": "[trigger link: Google review]",
  "nativeTags": [
    "review-queued",
    "review-requested",
    "review-reminded",
    "review-replied",
    "review-link-clicked"
  ],
  "webhook": {
    "action": "Custom Webhook",
    "event": "POST",
    "method": "POST",
    "url": "https://YOUR-N8N-DOMAIN/webhook/ghl-review-events",
    "authorization": "None",
    "headers": {
      "X-Review-Secret": "the value of REVIEW_WEBHOOK_SECRET"
    },
    "body": "Key/Value pairs, one row per field of the payload"
  },
  "payloads": {
    "request_sent": {
      "event": "request_sent",
      "locationId": "{{location.id}}",
      "contactId": "{{contact.id}}"
    },
    "reminder_sent": {
      "event": "reminder_sent",
      "locationId": "{{location.id}}",
      "contactId": "{{contact.id}}"
    },
    "customer_replied": {
      "event": "customer_replied",
      "locationId": "{{location.id}}",
      "contactId": "{{contact.id}}",
      "body": "{{message.body}}"
    }
  },
  "templates": {
    "request_sms": {
      "channel": "SMS",
      "text": "Hi {{contact.first_name}}, thank you for choosing Summit Roofing Co. If you have a minute, would you share your experience in a Google review? [trigger link: Google review] Reply STOP to opt out."
    },
    "reminder_sms": {
      "channel": "SMS",
      "text": "Hi {{contact.first_name}}, a quick reminder from Summit Roofing Co.: if you have a minute, a Google review of your experience helps other homeowners decide. [trigger link: Google review] Reply STOP to opt out."
    },
    "request_email": {
      "channel": "Email",
      "subject": "A quick favor from Summit Roofing Co.",
      "text": "Hi {{contact.first_name}},\n\nThank you for choosing Summit Roofing Co. If you have a minute, would you share your experience in a Google review? It helps other homeowners decide.\n\n[trigger link: Google review]\n\nThank you,\nThe Summit Roofing Co. team"
    },
    "reminder_email": {
      "channel": "Email",
      "subject": "A quick reminder from Summit Roofing Co.",
      "text": "Hi {{contact.first_name}},\n\nA quick reminder: if you have a minute, a Google review of your experience with Summit Roofing Co. helps other homeowners decide.\n\n[trigger link: Google review]\n\nThank you,\nThe Summit Roofing Co. team"
    }
  },
  "workflows": [
    {
      "key": "ask",
      "name": "Review Request 1: Ask after the job",
      "purpose": "Asks every customer for a Google review once the job is won or the appointment showed, then reminds them once.",
      "triggers": [
        {
          "trigger": "Opportunity Status Changed",
          "filters": {
            "In Pipeline": "your jobs pipeline (for example \"Roofing Jobs\")",
            "Moved To Status": "Won"
          }
        },
        {
          "trigger": "Appointment Status",
          "filters": {
            "In Calendar Filter": "your job or inspection calendar",
            "Filter By Appointment Status": "Showed"
          }
        }
      ],
      "settings": {
        "Allow Re-entry": "Off",
        "Stop on Response": "Off (the \"Replied or clicked?\" step stops the reminder for anyone who replied, so no setting lets a message from the customer, sent before or after the request, decide whether they get the request)",
        "Timezone": "Contact Timezone",
        "Time Window": "Specific Time on, Start Time 9:00 AM, End Time 6:00 PM, Include Days Monday to Saturday"
      },
      "steps": [
        {
          "type": "if_tags",
          "action": "If Else",
          "label": "Already asked?",
          "condition": "Contact Tags includes any of",
          "tags": [
            "review-requested",
            "review-queued"
          ],
          "then": [
            {
              "type": "end",
              "action": "Remove from Workflow",
              "label": "Stop here",
              "option": "Current Workflow"
            }
          ]
        },
        {
          "type": "add_tag",
          "action": "Add Contact Tag",
          "label": "Tag: queued",
          "tags": [
            "review-queued"
          ]
        },
        {
          "type": "wait",
          "action": "Wait",
          "label": "Wait 2 hours",
          "choice": "A set period of time",
          "duration": "2 hours",
          "advanceWindow": {
            "Resume On": "Mon, Tue, Wed, Thu, Fri, Sat",
            "Resume Between Hours": "9:00 AM to 6:00 PM"
          }
        },
        {
          "type": "add_tag",
          "action": "Add Contact Tag",
          "label": "Tag: requested",
          "tags": [
            "review-requested"
          ]
        },
        {
          "type": "if_phone",
          "action": "If Else",
          "label": "Has a phone number?",
          "condition": "Contact Details: Phone Is not empty",
          "then": [
            {
              "type": "send",
              "action": "Send SMS",
              "label": "Review request (SMS)",
              "template": "request_sms"
            },
            {
              "type": "webhook",
              "action": "Custom Webhook",
              "label": "Tell n8n: request sent",
              "payload": "request_sent"
            },
            {
              "type": "wait",
              "action": "Wait",
              "label": "Wait 3 days",
              "choice": "A set period of time",
              "duration": "3 days",
              "advanceWindow": {
                "Resume On": "Mon, Tue, Wed, Thu, Fri, Sat",
                "Resume Between Hours": "9:00 AM to 6:00 PM"
              }
            },
            {
              "type": "if_tags",
              "action": "If Else",
              "label": "Replied or clicked?",
              "condition": "Contact Tags includes any of",
              "tags": [
                "review-replied",
                "review-link-clicked"
              ],
              "then": [
                {
                  "type": "end",
                  "action": "Remove from Workflow",
                  "label": "Stop here",
                  "option": "Current Workflow"
                }
              ]
            },
            {
              "type": "add_tag",
              "action": "Add Contact Tag",
              "label": "Tag: reminded",
              "tags": [
                "review-reminded"
              ]
            },
            {
              "type": "send",
              "action": "Send SMS",
              "label": "Review reminder (SMS)",
              "template": "reminder_sms"
            },
            {
              "type": "webhook",
              "action": "Custom Webhook",
              "label": "Tell n8n: reminder sent",
              "payload": "reminder_sent"
            }
          ],
          "else": [
            {
              "type": "send",
              "action": "Send Email",
              "label": "Review request (email)",
              "template": "request_email"
            },
            {
              "type": "webhook",
              "action": "Custom Webhook",
              "label": "Tell n8n: request sent",
              "payload": "request_sent"
            },
            {
              "type": "wait",
              "action": "Wait",
              "label": "Wait 3 days",
              "choice": "A set period of time",
              "duration": "3 days",
              "advanceWindow": {
                "Resume On": "Mon, Tue, Wed, Thu, Fri, Sat",
                "Resume Between Hours": "9:00 AM to 6:00 PM"
              }
            },
            {
              "type": "if_tags",
              "action": "If Else",
              "label": "Replied or clicked?",
              "condition": "Contact Tags includes any of",
              "tags": [
                "review-replied",
                "review-link-clicked"
              ],
              "then": [
                {
                  "type": "end",
                  "action": "Remove from Workflow",
                  "label": "Stop here",
                  "option": "Current Workflow"
                }
              ]
            },
            {
              "type": "add_tag",
              "action": "Add Contact Tag",
              "label": "Tag: reminded",
              "tags": [
                "review-reminded"
              ]
            },
            {
              "type": "send",
              "action": "Send Email",
              "label": "Review reminder (email)",
              "template": "reminder_email"
            },
            {
              "type": "webhook",
              "action": "Custom Webhook",
              "label": "Tell n8n: reminder sent",
              "payload": "reminder_sent"
            }
          ]
        }
      ]
    },
    {
      "key": "replies",
      "name": "Review Request 2: Forward replies",
      "purpose": "Tags every reply from a contact who got the request and sends it to n8n. The tag is added here, for every reply, before n8n reads it.",
      "triggers": [
        {
          "trigger": "Customer Replied",
          "filters": {
            "Has Tag": "review-requested",
            "Doesn't Have Tag": "review-opted-out",
            "Reply Channel": "SMS"
          }
        },
        {
          "trigger": "Customer Replied",
          "filters": {
            "Has Tag": "review-requested",
            "Doesn't Have Tag": "review-opted-out",
            "Reply Channel": "Email"
          }
        }
      ],
      "settings": {
        "Allow Re-entry": "On (every reply must reach n8n)",
        "Stop on Response": "Off"
      },
      "steps": [
        {
          "type": "add_tag",
          "action": "Add Contact Tag",
          "label": "Tag: replied",
          "tags": [
            "review-replied"
          ]
        },
        {
          "type": "webhook",
          "action": "Custom Webhook",
          "label": "Send the reply to n8n",
          "payload": "customer_replied"
        }
      ]
    },
    {
      "key": "clicked",
      "name": "Review Request 3: Link clicked",
      "purpose": "Marks contacts who clicked the review link, so they get no reminder.",
      "triggers": [
        {
          "trigger": "Trigger Link Clicked",
          "filters": {
            "Trigger link": "Google review"
          }
        }
      ],
      "settings": {
        "Allow Re-entry": "Off"
      },
      "steps": [
        {
          "type": "add_tag",
          "action": "Add Contact Tag",
          "label": "Tag: link clicked",
          "tags": [
            "review-link-clicked"
          ]
        }
      ]
    }
  ]
}
```

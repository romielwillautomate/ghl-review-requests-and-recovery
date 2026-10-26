# Setup: GoHighLevel, Claude, Slack and n8n

You need a GoHighLevel sub-account with a phone number that can send SMS, self-hosted n8n reachable over HTTPS, an Anthropic API key and a Slack incoming webhook. The GoHighLevel workflows use the premium Custom Webhook action. Plan about 45 minutes, most of it for building the three GoHighLevel workflows.

Do the steps in this order: GoHighLevel basics, n8n, then the GoHighLevel workflows (they need the n8n URL).

## 1. GoHighLevel

### Private Integration token

1. Go to **Settings > Private Integrations > Create new integration**.
2. Give it these scopes:

   | Scope | Used for |
   |---|---|
   | `contacts.readonly` | Loading a contact, the digest's contact search, reading a contact's tasks |
   | `contacts.write` | Tags, DND after an opt out, the two send-time fields, recovery tasks and notes |
   | `conversations.readonly` | Finding the customer's conversation |
   | `conversations/message.readonly` | Reading the last messages for Claude |
   | `conversations/message.write` | Sending replies, and cancelling a reply that was scheduled for the morning |

   Two more are only needed once, for the lookups below: `locations/customFields.readonly` (custom field IDs) and `users.readonly` (the owner's user ID). You can leave them out and find the IDs another way.

3. Copy the token (it starts with `pit-`) into `GHL_API_KEY`.

### Location ID

**Settings > Business Profile** shows the Location ID. Put it in `GHL_LOCATION_ID`.

### Review link and trigger link

1. **Settings > Custom Values > New Custom Value**: name `Google Review Link`, value your Google review link. Your Google Business Profile gives you this link under "Ask for reviews". The key becomes `{{custom_values.google_review_link}}`.
2. **Marketing > Trigger Links > Links > Add Link** (older accounts: **Conversations > Trigger Links**): name `Google review`, Link URL: click `{}` and pick the Google Review Link custom value. The trigger link is what records a click per contact, which stops the reminder.

### Two custom fields for the send times

**Settings > Custom Fields**, object Contact, **Create field**, type single line text: `Review Request Sent` and `Review Reminder Sent`. n8n writes an ISO timestamp into them when GoHighLevel reports a request or a reminder, and the weekly digest counts from them. They are text fields because the API documents text values for custom fields.

Get their IDs with one call (scope `locations/customFields.readonly`) and copy the `id` of each into `GHL_FIELD_REQUEST_SENT_ID` and `GHL_FIELD_REMINDER_SENT_ID`:

```bash
curl "https://services.leadconnectorhq.com/locations/YOUR_LOCATION_ID/customFields?model=contact" \
  -H "Authorization: Bearer pit-your-token" -H "Version: 2021-07-28"
```

### The owner's user ID (for the recovery tasks)

The recovery task is assigned to `GHL_OWNER_USER_ID`, and the note is written as that user. Leave it empty and the task is unassigned. To find the ID (scope `users.readonly`):

```bash
curl "https://services.leadconnectorhq.com/users/?locationId=YOUR_LOCATION_ID" \
  -H "Authorization: Bearer pit-your-token" -H "Version: 2021-07-28"
```

HighLevel marks this call as deprecated in favor of `GET /users/search`, which also needs your agency's `companyId`. Either returns the users with their `id`.

Only text people who gave you permission to text them. In the US, sending through a GoHighLevel number also needs A2P 10DLC registration.

## 2. Anthropic

Create a key at **console.anthropic.com > API keys** and put it in `ANTHROPIC_API_KEY`. The default model is `claude-sonnet-5`; `claude-haiku-4-5` is cheaper. The request forces a tool call with thinking turned off, so `claude-opus-5-5` and the Claude Fable and Mythos models do not work: `claude-opus-5-5` and `claude-fable-5-1` refuse the forced tool call (checked with Anthropic's token counter), and the Fable and Mythos models and `claude-opus-5-5` refuse thinking turned off. The workflow refuses to start with one of them and names the problem.

## 3. Slack

Create an incoming webhook (**api.slack.com/apps > Create New App > Incoming Webhooks**) for the channel the owner reads, and put its URL in `SLACK_WEBHOOK_URL`. It gets the complaint, question and hand off alerts, the weekly digest and the error alerts. It is required: without it nobody would hear about complaints and questions, so the workflow stops at "Stop: Bad Settings" when it is empty.

## 4. n8n

1. Copy `.env.example` into your n8n environment, fill in the values and restart n8n. The nodes read them with `$env`, so self-hosted n8n needs `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. (n8n Cloud does not allow `$env`. There, replace `$env.` with `$vars.` and create the same names under Variables.)
2. Import `workflow/error-alert-slack.json` and **activate it**. n8n 2.x only runs an error workflow that is active; an inactive one is skipped without any alert.
3. Import `workflow/ghl-review-requests-and-recovery.json`. In its settings, choose the error alert workflow as its **Error Workflow**, and set **Timezone** to your business time zone (the weekly digest runs Mondays at 8 AM in it; change the day and hour in the **Schedule: Weekly** node).
4. Activate the workflow.

It now listens on two production URLs:

- `https://YOUR-N8N-DOMAIN/webhook/ghl-review-events` receives the GoHighLevel events (request sent, reminder sent, customer replied)
- `https://YOUR-N8N-DOMAIN/webhook/ghl-review-digest` posts the digest on demand

## 5. The GoHighLevel workflows

Build the three workflows from [ghl/native-workflows.md](../ghl/native-workflows.md): the review request with its one reminder, the reply forwarder, and the link click tag. The recipe has every trigger, filter, setting, action, message text and the Custom Webhook settings, with the shared secret in the `X-Review-Secret` header.

Turn off anything else that auto-replies to these customers (a Conversation AI bot, for example), or they will get two answers.

## 6. First test

1. Run the digest once to check the settings and the Slack webhook:

   ```bash
   curl -X POST "https://YOUR-N8N-DOMAIN/webhook/ghl-review-digest" \
     -H "X-Review-Secret: YOUR_REVIEW_WEBHOOK_SECRET" -H "Content-Type: application/json" -d '{}'
   ```

   You get `{"started":true}` right away, and Slack gets a digest with zeros.
2. Shorten both Wait steps in workflow 1 to 1 minute for the test (put them back afterwards). Create a contact with your own phone number in your jobs pipeline and mark the opportunity **Won**. A minute later you get the review text, the contact gets `review-requested`, and the contact's Review Request Sent field is filled. If the field stays empty, check the n8n execution list for a failed `request_sent` run.
3. Reply "Thanks, great job". You get a short thank-you and the contact gets `review-replied` and `review-reply-praise`.
4. From a second contact, reply with a complaint. Slack gets a service recovery alert, the owner gets a task, the contact gets a note and the apology.
5. From a third, reply "STOP" and check that DND for SMS is on.

## 7. Tags

With the default `TAG_PREFIX`:

| Tag | Added by | Meaning |
|---|---|---|
| `review-queued` | GoHighLevel workflow 1 | Started workflow 1. A second start stops at the guard |
| `review-requested` | GoHighLevel workflow 1 | Got the review request. Never asked again |
| `review-reminded` | GoHighLevel workflow 1 | Got the one reminder |
| `review-replied` | GoHighLevel workflow 2 | Replied at least once (any reply). No reminder |
| `review-link-clicked` | GoHighLevel workflow 3 | Clicked the review link. No reminder |
| `review-reply-praise`, `review-reply-complaint`, `review-reply-question`, `review-reply-other` | n8n | What Claude read in a reply |
| `review-needs-recovery` | n8n | Complained. The owner has a task, and the assistant stays quiet |
| `review-needs-human` | n8n | A person should answer. The assistant stays quiet. Remove the tag to hand the customer back |
| `review-opted-out` | n8n | Asked to stop. DND is on for that channel |

## Troubleshooting

Open the workflow's **Executions** list in n8n:

| You see | Likely cause |
|---|---|
| Run stopped at "Stop: Bad Secret" or "Stop: Bad Digest Secret" | The `X-Review-Secret` header (or `customData.secret`) does not match `REVIEW_WEBHOOK_SECRET` |
| Run stopped at "Stop: Bad Settings" or "Stop: Bad Digest Settings" | A setting is missing or invalid. The error lists every problem, for example an empty `GHL_FIELD_REQUEST_SENT_ID` or a wrong `BUSINESS_TZ` |
| A run failed but Slack stayed quiet | The error alert workflow is not active, is not chosen as the Error Workflow, or `SLACK_WEBHOOK_URL` is empty |
| Run ended at "Route Event" | The call was ignored on purpose: unknown event, no contact id, another location, or a channel other than SMS and email |
| Run ended at "Route Reply" | The reply was ignored on purpose: a repeat delivery, no `review-requested` tag, past `REPLY_WINDOW_DAYS`, already opted out, DND on, or a person already has the customer and Slack was told in the last 6 hours |
| Run stopped at "Stop: Record Failed" | The send time was not saved. Check the two field IDs and the `contacts.write` scope |
| Run stopped at "Stop: No Reply Text" | A reply arrived without its text. One picture-only message does this; answer it by hand. If every reply does, the `body` value of the Custom Webhook in workflow 2 is empty or not a merge field: pick `{{message.body}}` with the merge field picker |
| Run stopped at "Stop: Contact Lookup Failed" | `GHL_API_KEY` scopes (`contacts.readonly`) or a contact from another sub-account |
| "without the conversation history" | The customer was answered, but the conversation could not be loaded. Check the `conversations.readonly` and `conversations/message.readonly` scopes |
| Run stopped at "Stop: Opt Out Failed" | GoHighLevel did not accept the DND update. Set DND by hand right away |
| "could not be cancelled" after STOP | A reply was waiting for the morning and GoHighLevel refused to cancel it. Delete the scheduled message in the conversation |
| "The Slack alert could not be posted" | Slack refused the alert (a removed or disabled incoming webhook answers 404 `no_service`). The error has the whole alert, so act on it, then fix `SLACK_WEBHOOK_URL` |
| Run stopped at "Stop: Digest Failed" | A digest search or a task load failed, or Slack refused the post. Run the digest again with the curl call above once it is fixed |
| Customers who replied still get the reminder | Workflow 2 does not add `review-replied` (check its two Customer Replied triggers and the Add Contact Tag step), or "Replied or clicked?" in workflow 1 checks the wrong tags |
| A customer got two review requests | The first two steps of workflow 1 are missing or in the wrong order: "Already asked?" must check `review-requested` and `review-queued`, and "Tag: queued" must come before the 2 hour wait |

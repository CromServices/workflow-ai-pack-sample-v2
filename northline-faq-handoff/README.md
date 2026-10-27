# Northline Supplies: FAQ enquiry with human handoff

A sample workflow for n8n. A customer question comes in by webhook. If it matches one of six approved answers, the customer gets that answer straight away. If it doesn't, or the customer wants a person, a handoff note is emailed to your team and the customer is told a person will reply.

Fictional business; not a paid engagement.

Northline Supplies is a made-up Australian shop selling workshop consumables, fasteners, packaging and PPE. The policies in the answers (hours, delivery times, cut-off, free shipping threshold, returns) are sample figures, not real terms.

## What it does

1. Receives `POST {"question": "...", "name": "...", "email": "..."}`. Only `question` is required.
2. Cleans the text, and removes card-like numbers and pasted passwords or codes so they never reach the handoff email.
3. Looks the question up in six inline FAQ entries and scores each one.
4. If one entry clearly wins, replies with its answer.
5. Otherwise, writes a short handoff note (reason, the customer's words, reply address), emails it to your team, and replies "a person will reply".

It always hands off, and never guesses, when:

- nothing in the six entries matches, or two entries are too close to call
- the customer asks for a person, a call back or a manager
- the question needs someone to act: refunds, cancellations, damaged or missing items, wrong item, order status or an order number, address changes, quotes or discounts, trade accounts, legal or insurance matters, safety questions
- the customer pastes a card number, password or one-time code
- the question is empty

The reply never promises a refund, replacement or date.

## Nodes

| Node | Type | Job |
| --- | --- | --- |
| Enquiry webhook | Webhook (POST, responds from a later node) | Receives the enquiry |
| Normalise enquiry | Code | Cleans the text and removes sensitive details |
| FAQ knowledge source | Code | Holds the six FAQ entries and the handoff trigger words |
| Match question | Code | Scores the entries and decides answer or handoff |
| Confident match? | IF | Splits the two paths |
| Reply with answer | Respond to Webhook | Returns the matched answer |
| Write handoff note | Code | Builds the subject and body of the note |
| Send handoff email | Send Email (SMTP) | Emails the note to your team |
| Reply: a person will reply | Respond to Webhook | Tells the customer a person will reply |
| Reply: handoff failed | Respond to Webhook | If the email can't be sent, says so (HTTP 503) instead of claiming a person will reply |

Three sticky notes on the canvas explain how it works, the set-up steps and the fictional business.

## Import

1. In n8n choose **Create workflow**, then the **...** menu, then **Import from file**, and pick `northline-faq-handoff.workflow.json`. (Pasting the file's contents onto the canvas also works.)
2. The Send handoff email node shows a warning until you pick a credential.

## Configure

1. **SMTP credential.** Open Send handoff email and choose your own SMTP credential. The file holds a placeholder reference only; no credentials are included.
2. **Addresses.** Change `faq-desk@example.com` (from) and `handoff@example.com` (to) to your own. To use Slack or a ticket tool instead, replace this node and keep its error output connected to Reply: handoff failed.
3. **Knowledge.** Edit the `faq` list in FAQ knowledge source. Each entry has an `id`, a `topic`, a list of `keywords` (a phrase scores 2, a single word scores 1) and the approved `answer`. An entry needs at least 2 points and must beat the next best entry. Change `minScore` to be stricter or looser.
4. **Handoff triggers.** Edit `humanRequestPhrases` and `handoffTopicPhrases` in the same node. These words are checked first and always go to a person.
5. **Business name and hours.** The business name and hours also appear in the two Respond to Webhook nodes.
6. **Publish** the workflow, then send your site's enquiry form to the production webhook URL: `/webhook/northline-faq-enquiry` on your n8n address.

## Try it

```
curl -X POST https://YOUR-N8N-ADDRESS/webhook/northline-faq-enquiry \
  -H 'Content-Type: application/json' \
  -d '{"question":"How long does delivery take to Sydney?","email":"customer@example.com"}'
```

Answered: `{"status":"answered","topic":"Delivery times","answer":"...","source":"Northline Supplies FAQ"}`

Handed off: `{"status":"handed_off","message":"Thanks for your question. A person at Northline Supplies will reply, ...","reference":"ENQ-..."}`

## Limits

- This is keyword matching on six entries, not a language model. It is deliberately cautious: when it isn't sure, a person answers.
- It does not look up orders, take payments or change anything.
- It keeps no record of enquiries beyond n8n's own execution history. Turn execution saving off or shorten retention if you don't want customer text stored there.
- Tested on n8n 2.8.4 (see Test summary below).

## Test summary

Tested on n8n 2.8.4 with a local test mail server, using the production webhook URL of the imported, published workflow:

- 6 FAQ questions were each answered with the right entry.
- 6 handoff cases were each sent to a person with the right reason: no match, a request for a person, a refund request, a pasted card number, an order question with no reply address, and an empty question.
- A pasted card number was removed before the handoff email was written. It did not appear in the email.
- If the handoff email can't be sent, the reply is HTTP 503 (`handoff_failed`), not a promise that a person will reply.
- The workflow contains no credentials, no real email addresses and no personal data. Addresses are `example.com` placeholders.

## Files

| File | What it is |
| --- | --- |
| `northline-faq-handoff.workflow.json` | The workflow to import |
| `README.md` | This file |
| `shots/` | Screenshots: the canvas, a matched answer and a handoff |

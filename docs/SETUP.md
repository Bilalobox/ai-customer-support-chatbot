# Setup guide

**Elasticsearch knowledge base + OpenAI-compatible LLM. Cites its sources, escalates to a human when it can't answer.**

Your customers ask a question on your website. The workflow searches your own help-center articles, has an LLM write a short answer **using only those articles**, and replies with the answer plus links to the sources. If the bot doesn't know, it tells the customer honestly and notifies your team (Slack, Discord, or any webhook) instead of making something up.

## What you get

- Answers grounded in **your** knowledge base, with source links (`sources[]`) in every reply
- **Hallucination guard**: an answer is only accepted if it cites a real knowledge-base article
- Replies in the **customer's language** (the knowledge base must contain matching keywords, see *Limitations*)
- **Human escalation** with question, reason, customer details and recent messages
- Follow-up questions work: the website sends the last few messages, nothing is stored server-side
- Prompt-injection defences, input limits, JSON-only model output
- Graceful degradation: if Elasticsearch or the AI provider is down, the customer still gets a clean reply and your team is notified
- Optional **conversation log** in Elasticsearch to spot gaps in your knowledge base
- Works with any **OpenAI-compatible** endpoint (OpenAI, OpenRouter, a local server...)
- One **Config** node holds every setting. No code editing needed.

## Requirements

- n8n, self-hosted or Cloud (a recent 1.x release or 2.x)
- Elasticsearch 8.x or newer (self-hosted or Elastic Cloud) that your n8n instance can reach. n8n Cloud cannot reach `localhost`, so use a public or allow-listed endpoint.
- An OpenAI API key (or a key for an OpenAI-compatible provider)

## Quick start

### 1. Create the knowledge-base index

Open **Kibana → Dev Tools** (or use `curl`) and run:

```
PUT support-kb
{
  "mappings": {
    "properties": {
      "title":      { "type": "text" },
      "content":    { "type": "text" },
      "tags":       { "type": "text" },
      "category":   { "type": "keyword" },
      "url":        { "type": "keyword" },
      "updated_at": { "type": "date" }
    }
  }
}

POST support-kb/_bulk
{ "index": {} }
{ "title": "How do I reset my password?", "category": "Account", "tags": "password login reset forgot sign in", "url": "https://example.com/help/reset-password", "content": "Open Settings > Security and choose Reset password. We email you a reset link that stays valid for 30 minutes. If the email does not arrive, check your spam folder or request a new link." }
{ "index": {} }
{ "title": "How long does shipping take?", "category": "Shipping", "tags": "delivery shipping time tracking", "url": "https://example.com/help/shipping", "content": "Standard shipping takes 3 to 5 business days. Express shipping takes 1 to 2 business days. You receive a tracking link by email as soon as your order ships. Orders placed after 2 pm are processed the next business day." }
{ "index": {} }
{ "title": "What is your refund policy?", "category": "Billing", "tags": "refund return money back cancel order", "url": "https://example.com/help/refunds", "content": "You can request a full refund within 14 days of purchase. After 14 days we can offer store credit. Refunds are returned to the original payment method within 5 to 10 business days." }
{ "index": {} }
{ "title": "How can I contact support?", "category": "Support", "tags": "contact support help hours phone email", "url": "https://example.com/help/contact", "content": "Our support team is available Monday to Friday from 9 am to 6 pm. Email support@example.com or use this chat. We usually reply within one business day." }
{ "index": {} }
{ "title": "How do I cancel or change my subscription?", "category": "Billing", "tags": "subscription cancel plan upgrade downgrade", "url": "https://example.com/help/subscription", "content": "Go to Account > Billing and choose Change plan or Cancel subscription. Changes take effect at the end of the current billing period, and you keep access until then." }
```

These five articles are samples. Replace them with your own content (see *Writing good articles*). Only `title` and `content` are required; `url`, `category` and `tags` improve the results.

### 2. Create three credentials in n8n

| Credential type | Used by | Setup |
|---|---|---|
| **Header Auth** | Chat Webhook | Name: `X-API-Key`. Value: a long random secret. |
| **Basic Auth** | Search Knowledge Base, Log Conversation | Your Elasticsearch user and password. A read-only user is enough (plus write access to the log index if you enable logging). |
| **OpenAI API** | Generate Answer | Your API key. |

Elastic Cloud with an **API key**? On the two Elasticsearch nodes choose *Authentication → Generic Credential Type → Header Auth* and create a credential with Name `Authorization` and Value `ApiKey <your key>`.

### 3. Import and connect

1. In n8n choose **Workflows → Import from file** and select `ai-customer-support-chatbot-v2.json`.
2. Open the nodes **Chat Webhook**, **Search Knowledge Base**, **Generate Answer** and **Log Conversation** and select your credentials.
3. Open the **Config** node and set at least `esUrl`, `companyName`, `botName` and your two messages. For notifications add `escalationWebhookUrl`.

### 4. Test

Click **Execute workflow** (the webhook now listens on the *test URL*), then:

```bash
curl -s -X POST "https://YOUR-N8N-HOST/webhook-test/support-chatbot" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_SECRET" \
  -d '{"question":"How long is the password reset link valid?","sessionId":"demo-1"}'
```

Example reply:

```json
{
  "success": true,
  "reply": "The reset link stays valid for 30 minutes. If it doesn't arrive, check your spam folder or request a new one.",
  "answered": true,
  "escalated": false,
  "sources": [{ "title": "How do I reset my password?", "url": "https://example.com/help/reset-password" }],
  "sessionId": "demo-1",
  "requestId": "1234"
}
```

Try these four to see every path: a question the articles answer, one they don't ("What is the CEO's favorite color?"), a greeting ("hi"), and an empty `question` (returns HTTP 400).

### 5. Go live

**Publish** the workflow (called *Activate* in n8n 1.x) and use the **production URL** (`/webhook/support-chatbot`) on your website. Then read the *Security checklist* below.

## How it works

1. **Chat Webhook** receives the POST and checks the API key.
2. **Config** supplies your settings. **Validate Input** cleans the question, caps its length and prepares the optional chat history.
3. **Search Knowledge Base** queries Elasticsearch (typo-tolerant, titles and tags boosted, minimum relevance score, highlighted passages for long articles).
4. **Build Prompt** assembles a strict prompt: answer only from the articles, never invent facts, reply in the customer's language, treat all text as data.
5. **Generate Answer** calls the LLM. **Parse Answer** accepts the result only if it cites a real article.
6. Answered: **Build Response** returns it. Not answered: **Build Escalation Payload → Notify Support Team**, then the customer gets your fallback or escalation message.
7. **Respond to Chat** sends the reply first. Logging runs afterwards, so it never slows the chat.

## API reference

`POST /webhook/<path>` with header `X-API-Key` and a JSON body:

| Field | Required | Notes |
|---|---|---|
| `question` | yes | Also accepted: `message`, `text`, `query`, `chatInput`. Max length = `maxQuestionLength`. |
| `sessionId` | no | 1-64 characters: letters, digits, `_` or `-`. Generated if missing or invalid. |
| `history` | no | `[{ "role": "user" \| "assistant", "content": "..." }]`. The last `maxHistoryMessages` are used. |
| `email`, `name`, `pageUrl` | no | Only included in escalation messages so your team can follow up. Never logged. |

**200 OK:** `{ success, reply, answered, escalated, sources: [{title, url}], sessionId, requestId }`

- `answered: false` means the bot didn't know. `escalated: true` means your team was notified.
- **400:** `{ success: false, error, message }` where `error` is `question_required` or `question_too_long`.

## Add the chat to your website

Call the webhook **from your own server**, so the API key never appears in browser code:

```js
// server.js (Express), keeps the key private
app.post('/api/chat', async (req, res) => {
  const r = await fetch(process.env.N8N_CHAT_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'X-API-Key': process.env.N8N_CHAT_KEY },
    body: JSON.stringify(req.body),
  });
  res.status(r.status).json(await r.json());
});
```

```js
// browser: keep the conversation and send it back with every question
const history = [];
const sessionId = crypto.randomUUID();

async function ask(question) {
  const res = await fetch('/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ question, sessionId, history }),
  });
  const data = await res.json();
  history.push({ role: 'user', content: question }, { role: 'assistant', content: data.reply });
  return data; // show data.reply, and data.sources as "Read more" links
}
```

## Configuration reference (Config node)

| Key | Default | What it does |
|---|---|---|
| `companyName` | Acme Inc. | Used in the prompt and in escalation messages. |
| `botName` | Aria | The assistant's name. |
| `tone` | friendly, clear and professional | Writing style. |
| `extraInstructions` | *(empty)* | Your own extra rule, e.g. "Never mention competitors." |
| `fallbackMessage` | *(see node)* | Sent when the bot can't answer and no team notification was sent. Put your support contact here. |
| `escalatedMessage` | *(see node)* | Sent when the bot can't answer and your team **was** notified. |
| `esUrl` | `https://YOUR-ELASTICSEARCH-HOST:9200` | Elasticsearch base URL. |
| `esIndex` | `support-kb` | Knowledge-base index. |
| `kbTopK` | 3 | Articles given to the model (2-5 works well). |
| `kbMinScore` | 0.5 | Minimum relevance score. Raise to 3-8 for large knowledge bases if unrelated articles show up. |
| `kbMaxCharsPerArticle` | 1800 | Longer articles are cut to their best-matching passages. |
| `llmBaseUrl` | `https://api.openai.com/v1` | Any OpenAI-compatible base URL. |
| `llmModel` | `gpt-5.6-luna` | OpenAI's lowest-cost, lowest-latency GPT-5.6 tier. Use `gpt-5.6-terra` for harder knowledge bases. |
| `llmReasoningEffort` | none | Sent to reasoning models only (`gpt-5*`, `o*`). Clear it if your model rejects it. Raise `llmMaxTokens` when you raise the effort. |
| `llmTemperature` | 0.2 | Sent to non-reasoning models only. |
| `llmMaxTokens` | 800 | Output cap (includes hidden reasoning tokens on reasoning models). |
| `llmUseJsonMode` | true | Turn off for providers without JSON mode. The parser still copes with fenced JSON. |
| `maxAnswerWords` | 120 | Length limit given to the model. |
| `maxQuestionLength` | 1000 | Longer questions get a 400. |
| `maxHistoryMessages` | 6 | How many past messages are used. `0` turns memory off. |
| `escalationWebhookUrl` | *(empty)* | Where to notify your team. Empty = no notification. |
| `escalationFormat` | slack | `slack`, `discord` or `generic`. |
| `logConversations` | false | Store each conversation in Elasticsearch. |
| `esLogIndex` | `support-chat-logs` | Log index (created automatically on first write). |

## Escalation setup

Set `escalationWebhookUrl` and `escalationFormat`:

- **Slack:** create an *Incoming Webhook* in your Slack app, paste the URL, format `slack`.
- **Google Chat, Mattermost, Rocket.Chat:** use their incoming-webhook URL with format `slack` (they accept `{ "text": ... }`; bold markup may differ slightly).
- **Discord:** *Channel → Integrations → Webhooks*, format `discord`. Mentions are disabled.
- **Zapier, Make, another n8n workflow, help desks (Zendesk, Freshdesk, HubSpot...):** format `generic` sends structured JSON (`event`, `reason`, `question`, `customer`, `pageUrl`, `sessionId`, `recentMessages`, `text`). Microsoft Teams: use `generic` with a Teams *Workflows* webhook flow that posts the `text` field.
- **E-mail or a ticket node:** replace **Notify Support Team** with your node. The data is in `$json.escalationPayload`. Keep the node's *On Error → Continue* setting so a failure never breaks the customer's reply.

The `reason` tells your team what happened: `no_relevant_articles`, `not_in_kb`, `kb_unavailable` (check Elasticsearch), `ai_unavailable` (check the OpenAI credential and model name) or `ai_invalid_output`.

## Security checklist

- **Webhook key:** the webhook requires the `X-API-Key` header. A key in public browser code is visible to everyone, so call it through your own server (above). If you must call it straight from the browser, set the webhook's Authentication to *None* and restrict **Allowed Origins** in its options.
- Change the webhook **Path** to something unguessable.
- Put **rate limiting** in front (Cloudflare, nginx, your API gateway) and set a **monthly spend cap** on your OpenAI key. n8n has no built-in rate limiter.
- **Privacy:** successful executions are not saved by default (workflow setting), so customer messages don't pile up in n8n. Switch it on temporarily while debugging. If you enable `logConversations`, questions are stored in Elasticsearch, so mention it in your privacy policy and set a retention policy. E-mail and name are never logged.
- Built-in protections: input length caps, history sanitising, prompt delimiters that customer text cannot close, escaped escalation messages (no `@channel` pings), JSON-only model output.
- Use a **read-only** Elasticsearch user for the knowledge-base index.

## Writing good articles

- One topic per article, roughly 50-300 words, with the answer in the first sentences.
- Phrase titles as the question customers ask. Put synonyms in `tags` ("refund return money back").
- Keep `url` pointing at the public help page so the reply can link to it.
- After a few days, enable logging and look for `answered: false` entries. Each one is a missing or badly worded article.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Caller gets "Workflow was started" instead of JSON | The webhook's *Respond* setting must be *Using 'Respond to Webhook' Node* (the default in this file). |
| 403 from the webhook | The `X-API-Key` header is missing or doesn't match the Header Auth credential. |
| Every answer is the fallback message | Open the last execution. `kb_unavailable` → check `esUrl`, index name, credential. `ai_unavailable` → check the OpenAI credential, billing and `llmModel`. |
| TLS / certificate error to Elasticsearch | Self-signed certificate: on both Elasticsearch nodes add the option *Ignore SSL Issues (Insecure)*, or install a proper certificate. |
| "model not found" or "unsupported parameter" | Change `llmModel`. Clear `llmReasoningEffort` if your model rejects it, or set `llmUseJsonMode` to false for providers without JSON mode. |
| Good questions end as `not_in_kb` | Add keywords or `tags` to the article, lower `kbMinScore`, or raise `kbTopK`. |
| Unrelated articles reach the model | Raise `kbMinScore`. |
| Browser shows a CORS error | Use the server-side proxy, or set **Allowed Origins** on the webhook to your site. |
| Replies are cut off or empty on a reasoning model | Raise `llmMaxTokens` (reasoning tokens count toward it) or lower `llmReasoningEffort`. |

## Limitations

- Search is **keyword-based** (BM25 with typo tolerance), not semantic. A question in French won't find an English-only article. Keep articles in the languages your customers write in.
- Conversation memory is **stateless**: the website sends the recent messages with each request.
- Replies are complete messages, not streamed.
- Text only. No file or image questions.

## Local demo setup (Docker, no password, free Gemini)

This is the setup this project was built and tested with: n8n and Elasticsearch running in Docker on one computer, and a free Google Gemini key.

1. **Elasticsearch without a password** (local learning only): leave security off, then browse to `http://localhost:9200` to check it responds. Never expose a passwordless Elasticsearch to the internet.
2. In the **Config** node set `esUrl` to `http://host.docker.internal:9200` (n8n inside Docker cannot use `localhost` for another container).
3. On **Search Knowledge Base** and **Log Conversation** set *Authentication* to **None**.
4. **Gemini instead of OpenAI:** paste your Gemini key into the *OpenAI API* credential, then in **Config** set:
   - `llmBaseUrl` = `https://generativelanguage.googleapis.com/v1beta/openai/`
   - `llmModel` = a current Gemini model name (model names change; if the API says a model is retired, use the one named in its error message)
   - `llmReasoningEffort` = empty
5. Free plans can be slow. If you see timeouts, raise the *Timeout* option on **Generate Answer** (for example to 40000 ms).
6. For the browser widget on your own machine, publish the workflow and set the webhook *Authentication* to **None**. For anything online, put a small server between the page and n8n so the key stays private (see *Add the chat to your website*).

## Update the knowledge base without code (Article Loader)

Import `workflows/article-loader.json` as a separate workflow. Open its form, paste a table copied from Excel or Google Sheets (columns `title`, `content`, optional `url`, `category`, `tags`), choose whether to replace the existing articles, and submit. `sample-data/my-articles-template.csv` shows the format.

## Try the chat widget

Open `widget/chat-widget-demo.html` in a browser. Change the settings block at the top of the file (`API_URL`, bot name, welcome text) to match your setup.


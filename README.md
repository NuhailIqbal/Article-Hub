# Article Hub — AI Article Generation System (n8n)

An AI article-generation system built in n8n: single-article and bulk generation with live web research, SEO-ready output, and AI-generated images — the pipeline behind onboarding a client onto a content platform.

Built by [Nuhail Iqbal](https://www.linkedin.com/in/nuhail-iqbal-a93b1935b/) — AI Agent & Automation Engineer (n8n · VAPI · RAG · WhatsApp Chatbots · Python).

## How it works

Three webhooks, one system:

### 1. Article generation — `POST /webhook/article-generate`

```
Webhook → If (route) → AI Agent → Prepare Image Jobs → If Images Requested
  → Generate Image → Collect Generated Images → Insert Images Into Article → Respond
  ↘ (no images) Return Article Without Images → Respond
```

- **Webhook** receives a content brief: topic, style, settings, and the client profile with internal links.
- **AI Agent** (LangChain agent, Claude Opus 4.6 via OpenRouter) writes a humanized, SEO-optimized, AI-search-ready article from the brief.
- **Search the web in Perplexity** is attached as a tool, so the article is grounded in live research instead of model memory alone.
- **Prepare Image Jobs / Generate Image with OpenRouter** builds up to 10 image prompts from the article and generates them through OpenRouter.
- **Collect Generated Images / Insert Images Into Article** places the images into the finished article; if no images were requested, the article is returned as-is.

### 2. Image generation — `POST /webhook/article-image`

A standalone image endpoint: prompt + style + aspect ratio in, generated images out. Used by the article flow and on its own.

### 3. Bulk generation — `POST /webhook/article-bulk`

```
Webhook → Respond Immediately (202) → Explode Topics → Call Article Webhook (per topic)
  → Build Callback Payload → Save Article Callback
```

- **Webhook** accepts a bulk payload: a list of topics, global settings, and the client profile.
- **Respond Immediately** returns `202 Accepted` with the batch ID, so the caller never waits on generation.
- **Explode Topics** expands the batch into one full article request per topic, in the exact shape the article webhook expects.
- **Call Article Webhook** runs each topic through the single-article flow.
- **Build Callback Payload / Save Article Callback** sends the finished article back to your API for storage.

## Client onboarding in one line

A client's topics, profile, and internal links go in; publish-ready, researched, illustrated articles come out — one at a time, or a whole batch at once.

## Try it

### Prerequisites

- n8n (self-hosted or cloud)
- An OpenRouter API key (Claude Opus 4.6 is the configured model)
- A Perplexity API key (live web research tool)
- An API endpoint that accepts finished articles (the save callback)

### Setup

1. Import `workflows/Article_Hub.json` into n8n (Workflows → Import from file).
2. Create credentials in n8n and connect them to the matching nodes:
   - OpenRouter account → both **OpenRouter Chat Model** nodes
   - Perplexity account → **Search the web in Perplexity**
3. Set the two placeholder values in the HTTP nodes:
   - `YOUR-N8N-HOST` → your n8n host (used by the bulk flow to call the article webhook)
   - `YOUR-API-HOST` and `YOUR_WEBHOOK_SECRET` → your API host and shared webhook secret (used by the save callback)
   - `YOUR_OPENROUTER_API_KEY` → your OpenRouter key (Authorization header on the image-generation requests)
4. Activate the workflow and generate an article:

```bash
curl -X POST "https://YOUR-N8N-HOST/webhook/article-generate" \
  -H "Content-Type: application/json" \
  -d '{
    "contentBrief": {
      "topic": "How to choose an SEO agency",
      "style": "professional",
      "shouldGenerateImages": true
    }
  }'
```

Bulk generation:

```bash
curl -X POST "https://YOUR-N8N-HOST/webhook/article-bulk" \
  -H "Content-Type: application/json" \
  -d '{
    "topics": [{"topic": "Topic one"}, {"topic": "Topic two"}],
    "globalSettings": {},
    "clientProfile": {},
    "meta": {"batchId": "batch-001"}
  }'
```

## What this demonstrates

- Multi-agent article production: writing, live research, and image planning as separate, testable steps
- Async bulk processing — the caller gets a `202` immediately while generation runs
- Grounded output — Perplexity research feeds the writer instead of relying on the model's cutoff
- A real callback contract — finished articles are posted back to your API, not left in the workflow

## Files

| File | What it is |
| --- | --- |
| `workflows/Article_Hub.json` | The importable n8n workflow (27 nodes, 3 webhooks) |

## Notes

- The workflow ships with `active: false` and no credentials — add your own in n8n after import.
- All internal hosts, secrets, and product names from the original deployment have been replaced with placeholders.

## License

MIT — see [LICENSE](LICENSE).

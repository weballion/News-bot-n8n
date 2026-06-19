# News Bot — AI-Powered News Pipeline & Content Studio

An automated news aggregation and content production system built with n8n. Parses RSS feeds, generates longread articles, and provides a Telegram-based content studio with AI copywriting, image generation, and image editing — all orchestrated through natural language commands.

---

## Architecture

```
                          ┌──────────────────────┐
                          │  NEWS_Parser          │
                          │  (scheduled)          │
                          │                       │
                          │  RSS feeds → scrape → │
                          │  deduplicate → save   │
                          └──────────┬────────────┘
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │  NEWS_Longread        │
                          │                       │
                          │  AI analysis →        │
                          │  AI longread →        │
                          │  GitHub Gist →        │
                          │  Telegram notify      │
                          └──────────────────────┘


┌───────────┐     ┌──────────────────────────────────────────────────┐
│ Telegram  │────▶│  NEWS_Qualifizer                                 │
│           │     │                                                  │
│ text      │     │  voice/text/photo/forward → AI intent classifier │
│ voice     │     │  → create/update task → dispatch to department   │
│ photo     │     │                                                  │
│ forward   │     └──────┬──────────┬──────────┬──────────┬─────────┘
└───────────┘            │          │          │          │
                         ▼          ▼          ▼          ▼
                   ┌──────────┐ ┌────────┐ ┌────────┐ ┌────────┐
                   │Copywrite │ │Copywrite│ │ Image  │ │ Image  │
                   │  (HTML)  │ │  (MD)   │ │  Edit  │ │  Gen   │
                   └────┬─────┘ └───┬────┘ └───┬────┘ └───┬────┘
                        │           │          │          │
                        ▼           ▼          ▼          ▼
                   ┌──────────────────────────────────────────────┐
                   │              Supabase                        │
                   │  tasks · sent_messages                       │
                   └─────────────────────────────────────────────┘
```

---

## Workflows

### 1. NEWS_Parser — RSS Aggregator

Scheduled workflow that collects news from multiple sources.

**Trigger:** Schedule (configurable interval).

**Flow:**

1. Fetch news sources from Airtable.
2. Read RSS feeds for each source one by one.
3. Filter by freshness (last 24 hours) and validate dates.
4. Fetch already-processed links from Airtable to deduplicate.
5. Scrape full article content via ScrapingBee for new links (up to 8 per run).
6. Clean and structure the scraped data.
7. Save processed articles to Airtable.
8. Aggregate results and pass them to NEWS_Longread.
9. Send progress stats to Telegram (links found, filtered, saved).

---

### 2. NEWS_Longread — AI Article Writer

Receives parsed articles from the Parser and produces a longread analysis.

**Trigger:** Called by NEWS_Parser as a sub-workflow.

**Flow:**

1. Prepare article data for the AI Analyst agent.
2. AI Analyst evaluates the articles (relevance, trends, key insights).
3. AI Longread Writer generates a structured article based on the analysis.
4. Publish the article as a GitHub Gist.
5. Send a summary with the Gist link to Telegram.
6. Save results to Supabase (`sent_messages`) and Airtable.

---

### 3. NEWS_Qualifizer — Telegram Command Center

The main Telegram bot that acts as a content production dispatcher. Accepts natural language commands, classifies intent with AI, and routes tasks to specialized workflows.

**Trigger:** Telegram messages (text, voice, photo, forwarded messages).

**Flow:**

1. Receive a message → check user authorization.
2. Route by message type: voice → transcribe (Whisper) → text; photo → save; forwarded → extract content; command → handle `/start`, `/stop`.
3. Load context: check if the message is a reply, load active tasks and recent messages from Supabase.
4. AI Agent classifies intent: `new_task`, `task_update`, `dialogue`, or `clarification`.
5. For new tasks: create a task in Supabase with extracted parameters (department, style, content).
6. For task updates: update the existing task.
7. Dispatch to the appropriate department workflow based on task type:
   - `copywriting` → Copywriting (HTML or Markdown)
   - `image_gen` → Image Generation
   - `image_edit` → Image Editing
   - `video_gen` / `research` → (reserved for future workflows)
8. Send confirmation to Telegram.

**Supported departments:** copywriting, image_gen, image_edit, video_gen (planned), research (planned).

---

### 4. NEWS_Copywriting_HTML — AI Copywriter (HTML output)

Generates formatted content in HTML for web publishing.

**Trigger:** Called by Qualifizer as a sub-workflow.

**Flow:**

1. Load task data from Supabase.
2. Prepare context and set style preferences.
3. AI Agent writes content using SerpAPI (web search) and Jina AI (page reader) for research.
4. Validate output structure.
5. Send the result to Telegram.
6. Save to `sent_messages` and update task status.

---

### 5. NEWS_Copywriting_MD — AI Copywriter (Markdown output)

Same pipeline as the HTML copywriter, but outputs Telegram-optimized Markdown using the telegramify-markdown plugin.

---

### 6. NEWS_Image — AI Image Editor

Edits existing images based on text instructions.

**Trigger:** Called by Qualifizer as a sub-workflow.

**Flow:**

1. Load task data and determine the source photo.
2. Download the original image from Telegram.
3. AI Vision analyzes the photo and enhances the editing prompt.
4. Upload to imgbb (temporary hosting for the API).
5. Send to fal.ai Nano Banana API for editing.
6. Poll for completion, download the result.
7. Send the edited image to Telegram.
8. Save to `sent_messages` and update task status.

---

### 7. NEWS_Image_Gen — AI Image Generator

Generates images from text prompts, optionally referencing a previous image for style consistency.

**Trigger:** Called by Qualifizer as a sub-workflow.

**Flow:**

1. Load task data and check for a reference task/image.
2. If a reference image exists: AI Vision analyzes it to extract style. If not: text-only prompt generation.
3. Generate the image via fal.ai Flux Pro v1.1 Ultra.
4. Poll for completion, download the result.
5. Send the generated image to Telegram.
6. Save to `sent_messages` and update task status.

---

### 8. NEWS_ReAct — Reserved

Placeholder workflow for a future ReAct agent. Currently contains only a sub-workflow trigger.

---

## Tech Stack

| Component | Technology |
|---|---|
| Orchestration | n8n (8 linked workflows) |
| LLM | OpenRouter (multi-model) |
| Vision | OpenAI Vision API |
| Image generation | fal.ai Flux Pro v1.1 Ultra |
| Image editing | fal.ai Nano Banana |
| Image hosting | imgbb (temporary upload for API) |
| Web scraping | ScrapingBee |
| Web search | SerpAPI |
| Page reading | Jina AI Reader |
| Voice transcription | OpenAI Whisper |
| Article publishing | GitHub Gists |
| Database | Supabase (PostgreSQL) |
| Content sources | Airtable (RSS source list + processed articles) |
| Messenger | Telegram Bot API |
| Markdown conversion | telegramify-markdown (n8n plugin) |

---

## Database Schema

### Supabase

| Table | Purpose |
|---|---|
| `tasks` | Task queue — stores content requests with type, status, parameters, and department |
| `sent_messages` | Archive of all generated content sent to Telegram |

**Custom RPC:** `find_context_by_reply` — resolves Telegram reply chains to find the original task context.

### Airtable

| Table | Purpose |
|---|---|
| Sources | RSS feed URLs and metadata |
| Processed | Already-scraped article links for deduplication |

---

## Setup

### Prerequisites

- A running n8n instance (self-hosted or cloud).
- API keys for all services listed below.
- A Supabase project with `tasks` and `sent_messages` tables and the `find_context_by_reply` RPC function.
- An Airtable base with source and processed article tables.
- A Telegram bot created via @BotFather.

### Installation

1. Import all 8 JSON files into n8n.
2. Create the required credentials (see table below).
3. Wire the sub-workflow IDs: in NEWS_Qualifizer, update the Execute Workflow nodes (44–48) to point to your imported workflow IDs for Copywriting, Image Gen, Image Edit, etc. Same for NEWS_Parser → NEWS_Longread.
4. Activate workflows in order: sub-workflows first (Longread, Copywriting, Image, Image Gen, ReAct), then Parser and Qualifizer last.

### Credentials

| Credential | Workflows | Purpose |
|---|---|---|
| Telegram Bot API | Qualifizer, Copywriting, Image, Image Gen, Longread, Parser | Messaging |
| OpenRouter API | Qualifizer, Copywriting, Image, Image Gen, Longread | LLM (multi-model) |
| OpenAI API | Qualifizer | Whisper transcription |
| Supabase API | Qualifizer, Copywriting, Image, Image Gen, Longread | Database |
| Airtable API | Parser, Longread | News sources & archive |
| ScrapingBee API | Parser | Web scraping |
| SerpAPI | Copywriting (HTML & MD) | Web search for research |
| Jina AI | Copywriting (HTML & MD) | Page content extraction |
| fal.ai API | Image, Image Gen | Image generation & editing |
| imgbb API | Image | Temporary image upload |
| GitHub Token | Longread | Gist publishing |
| HTTP Header Auth | Qualifizer | Telegram typing indicator |

---

## Security

Most sensitive data has been replaced with `YOUR_*` placeholders. Review before publishing:

| What | File | Risk | Action |
|---|---|---|---|
| Bot user ID `8307189377` in URL | Qualifizer | **Medium** — reveals the bot's Telegram ID | Replace entire URL |
| Author name `Semenovych Oleksii` | ReAct (`authors` field) | **Medium** — personal data | Replace with placeholder |
| Supabase project URL `dwuekivjgvxbppjcltpu.supabase.co` | Qualifizer | **Medium** — reveals the Supabase instance | Replace with `YOUR_SUPABASE_URL` |
| Credential IDs (e.g. `08rilHeqmVcSA8cN`) | all | Low — n8n internal | Optional cleanup |
| Project ID `59YaPd0amPnB6Dlx` | all | Low — n8n internal | Optional cleanup |

### Quick fix

```bash
# Telegram bot ID (Qualifizer)
sed -i 's|https://api.telegram.org/[^"]*/sendChatAction|https://api.telegram.org/YOUR_BOT_TOKEN/sendChatAction|g' \
  NEWS_Qualifizer_*.json

# Author name (ReAct)
sed -i 's/Semenovych Oleksii/YOUR_NAME/g' NEWS_ReAct_*.json

# Supabase project URL (Qualifizer)
sed -i 's|dwuekivjgvxbppjcltpu.supabase.co|YOUR_SUPABASE_PROJECT.supabase.co|g' \
  NEWS_Qualifizer_*.json

# Optional: clean all credential and project IDs
for f in NEWS_*.json MEWS_*.json; do
  sed -i 's/59YaPd0amPnB6Dlx/YOUR_PROJECT_ID/g' "$f"
done
```

---

## License

MIT

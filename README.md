# AIH Content Factory v1.0 — n8n Workflow

One file: **`AIH_Content_Factory.json`**. Import it into n8n (Workflows → Import from File) and all 7 workflows from your PDR are already connected as a single pipeline (23 nodes). Nothing to wire manually — you only need to plug in credentials (below) and hit **Activate**.

---

## 1. What you must configure before it runs

n8n cannot know your API keys/OAuth accounts, so every node that talks to an external service is pre-built but needs YOU to attach credentials. Nothing else needs to change.

| # | Node name in canvas | What to do |
|---|---|---|
| 1 | **01 - Telegram Trigger** | Create a Telegram credential (`Telegram API`) using your bot token from @BotFather. Attach it here. |
| 2 | **05 - AI Research Engine (LLM Call)** | This is your **LLM connector**. It's built as a generic HTTP Request (not tied to one AI vendor), so you can point it at Claude, GPT-5.5, or Gemini. See Section 2 below. |
| 3 | **07 - AI Blog Writer (LLM Call)** | Same LLM connector as above (uses the same env vars). |
| 4 | **09 - AI SEO Engine (LLM Call)** | Same LLM connector as above. |
| 5 | **11 - AI Social Media Engine (LLM Call)** | Same LLM connector as above. |
| 6 | **16 - Create/Get Year Folder** | Google Drive OAuth2 credential. |
| 7 | **17 - Create/Get Month Folder** | Same Google Drive credential. |
| 8 | **18 - Create Google Doc** | Google Docs OAuth2 credential. |
| 9 | **19 - Insert Doc Content** | Same Google Docs credential. |
| 10 | **21 - Send Success Notification** | Same Telegram credential as node 1. |
| 11 | **23 - Send Error Notification** | Same Telegram credential as node 1. |

After import, n8n will show a red warning icon on these 11 nodes until you select/create the credential in each one (click the node → Credential dropdown → "Create New").

---

## 2. The LLM connector — where "GPT-5.5 / Gemini" plugs in

Per your PDR (Section 7, LLM = GPT-5.5 / Gemini), the AI nodes are built as **generic HTTP Request nodes** instead of a vendor-locked node, so you can swap providers without rebuilding the workflow.

They read **3 environment variables** (set these in n8n → Settings → Environment Variables, or your n8n `.env` file):

```
AIH_LLM_API_URL   = https://api.anthropic.com/v1/messages      (default, already set)
AIH_LLM_API_KEY   = your-api-key-here
AIH_LLM_MODEL      = claude-sonnet-4-6                          (or your preferred model)
```

- **Using Claude/Anthropic (default):** just set `AIH_LLM_API_KEY`. Nothing else to change.
- **Using OpenAI (GPT-5.5):** change `AIH_LLM_API_URL` to `https://api.openai.com/v1/chat/completions`, set `AIH_LLM_API_KEY` to your OpenAI key, set `AIH_LLM_MODEL` to `gpt-5.5`. You'll also need to edit the `jsonBody` in the 4 AI nodes — OpenAI's request shape (`messages` with `role: system`) is slightly different from Anthropic's; the `system` field moves inside the `messages` array as a `{role:'system', content:...}` entry instead of a top-level key.
- **Using Gemini:** change `AIH_LLM_API_URL` to the Gemini `generateContent` endpoint (includes your key as a query param), and edit the same 4 nodes' body to Gemini's `contents`/`parts` shape.

The 4 AI nodes are: **05 - AI Research Engine**, **07 - AI Blog Writer**, **09 - AI SEO Engine**, **11 - AI Social Media Engine**. Each has retry-on-fail set to 2 attempts, matching your PDR's error-handling spec (Section 15).

---

## 3. What each node does (in pipeline order)

**Workflow 01 — Telegram Trigger**
1. **01 - Telegram Trigger** — Listens for a message on your Telegram bot (URL, topic text, or an uploaded PDF).
2. **02 - Build Standard Job Payload** — Detects whether the input is a URL, a plain topic, or a PDF; generates a Job ID; builds the empty `{job, research, article, seo, social, thumbnail, schema, output}` payload from PDR Section 8.

**Workflow 02 — Research Engine**
3. **03 - Fetch Webpage Content** — If the input was a URL, downloads the raw page text. If it was a topic/PDF, this call is skipped gracefully (no error stops the pipeline).
4. **04 - Merge Webpage Content Into Payload** — Attaches the fetched text (or nothing) into `research.rawWebpageContent`.
5. **05 - AI Research Engine (LLM Call)** — Sends the raw content to your LLM with a system prompt that enforces the source-priority order from your PDR (Official Notification → Official PDF → Official Website → Government Portal → Trusted Government Source → Input URL), and asks for a structured Verified Research JSON.
6. **06 - Parse Research JSON** — Parses the LLM's text response into real JSON and merges it into `job.research`.

**Workflow 03 — Blog Writer**
7. **07 - AI Blog Writer (LLM Call)** — Sends the verified research JSON to the LLM using your full "AIH Master Prompt" rules (Telugu+English tone, all required sections: eligibility, vacancy, salary, FAQs, etc.).
8. **08 - Parse Article JSON** — Parses the returned article JSON into `job.article`.

**Workflow 04 — SEO Engine** (runs in parallel with Workflow 05 below)
9. **09 - AI SEO Engine (LLM Call)** — Generates SEO title, meta description, slug, keywords, tags, schema markup, Rank Math fields.
10. **10 - Parse SEO JSON** — Parses that into `job.seo` / `job.schema`.

**Workflow 05 — Social Media Engine** (runs in parallel with Workflow 04)
11. **11 - AI Social Media Engine (LLM Call)** — Generates distinct captions for Instagram, Facebook, Telegram, WhatsApp, LinkedIn, Threads, Twitter/X, Pinterest, YouTube description, YouTube community post.
12. **12 - Parse Social Media JSON** — Parses that into `job.social`.
13. **13 - Merge SEO + Social Branches** — Rejoins the two parallel branches (n8n Merge node).
14. **14 - Consolidate Full Payload** — Combines both branch outputs back into one single job object.

**Workflow 06 — Google Docs Generator**
15. **15 - Assemble Google Doc Content** — Builds one long formatted document string (SEO block, full article, thumbnail prompt, all social captions, JSON-LD schema, and the quality checklist from PDR Section 16).
16. **16 - Create/Get Year Folder** — Creates (or reuses) a Google Drive folder named after the current year.
17. **17 - Create/Get Month Folder** — Creates (or reuses) a subfolder named after the current month, inside the year folder.
18. **18 - Create Google Doc** — Creates a new Google Doc titled from the article's hero title, inside the month folder.
19. **19 - Insert Doc Content** — Writes the fully assembled content (from node 15) into that Google Doc.
20. **20 - Attach Doc Link To Payload** — Grabs the created Doc's URL and stores it on the job payload.

**Workflow 07 — Telegram Notification**
21. **21 - Send Success Notification** — Sends you a Telegram message confirming Job ID, article title, whether the source was officially verified, and the Google Doc link.

**Error handling (Section 15 of your PDR)**
22. **22 - Format Error Message** — Catches failures from any AI call, either Google Docs step, or the success-notification step (after 2 retries each), and builds a readable error message with node name, error, and timestamp.
23. **23 - Send Error Notification** — Sends that error message to you on Telegram, so a failed job never disappears silently.

---

## 4. Notes / things you may want to adjust

- **Google Drive root folder:** by default new Year folders are created at "My Drive" root. To target a specific parent folder, set an environment variable `AIH_GDRIVE_ROOT_FOLDER_ID` to that folder's ID.
- **PDF input handling:** the payload builder (node 02) detects a PDF upload and records its Telegram `file_id`, but downloading/OCR-ing that PDF isn't wired in yet — your PDR excludes automatic file parsing from V1 scope implicitly by not specifying it, and I didn't want to invent a section that wasn't in your spec. If you want it, I can add a "Download Telegram File → Extract PDF Text" step feeding into node 04 — say the word and I'll add it.
- **Excluded by design (per your PDR Section 18 — Future Versions):** Auto WordPress Publish, Auto Thumbnail Generation, Auto Social Posting, AI Memory, RAG, Vector DB, Analytics Dashboard, Approval Workflow. This build intentionally stops at "Google Doc ready for manual WordPress publishing," matching your V1.0 scope exactly.
- **Category/keyword logic:** the AI prompts already encode your Section 4 category list, so the SEO/Research nodes will classify content into one of your 13 defined categories rather than inventing new ones.

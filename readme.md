# How to Import and Run the Agentic Crypto News Workflow in n8n

This guide walks you through **importing**, **configuring**, **testing**, and **scheduling** the `n8n-workflow.json` that powers the Agentic n8n Pipeline for Weekly Crypto Intelligence.

> Works with n8n Cloud and self‑hosted n8n (Docker/Node). No code changes required—only credentials and a few node fields.

---

## 1) Import the Workflow

1. Open your n8n UI and sign in.
2. Go to **Workflows → Import from File**.
3. Select your local copy of **`n8n-workflow.json`**.
4. Click **Import**. The workflow will appear in your list (status: *Not active*).

> Tip: Rename the workflow to something clear (e.g., `Crypto Weekly – Agentic Publisher`).

---

## 2) Create/Configure Credentials

Open **Credentials** in n8n and create the following:

### A) SerpApi
- **Type**: HTTP Request (or dedicated SerpApi credential if available)
- **Base URL**: `https://serpapi.com`
- **API Key**: Your SerpApi key
- The workflow’s news node queries **Google News** with `q="Crypto News this week"`

### B) OpenAI (Text Models)
- **Type**: OpenAI
- **API Key**: Your OpenAI API key
- **Model (first compose node)**: `gpt-4.1-mini`
- **Model (translation/newsletter)**: `chatgpt-4o-latest` (as specified in the workflow)

### C) OpenAI (Image Generation)
- Reuse the **OpenAI** credential
- Ensure your account has image generation enabled

### D) WIX (Blog + Media)
- **Type**: HTTP Request
- **Auth**: API Key (Bearer) or OAuth, depending on your Wix setup
- **Required APIs**:
  - **Blog Draft Posts API**: `POST https://www.wixapis.com/blog/v3/draft-posts`
  - **Site Media API (Generate Upload URL)**: `POST https://www.wixapis.com/site-media/v1/files/generate-upload-url`
- Ensure your Site ID/App has permissions to Blog + Media

### E) SendGrid
- **Type**: HTTP Request (or SendGrid credential if available)
- **Base URL**: `https://api.sendgrid.com/v3`
- **API Key**: Your SendGrid key
- **Single Send endpoint** (used by the workflow): `/marketing/singlesends`

> **Security tip**: Use n8n Credentials rather than hardcoding secrets in nodes.

---

## 3) Node Fields to Review/Adjust

Open the imported workflow and check these nodes (names may vary slightly):

1. **SerpApi – Google_news search**
   - Query: `Crypto News this week`
   - Add/adjust `gl`, `hl`, or freshness params if needed

2. **Message a model (gpt‑4.1‑mini)**
   - Produces two outputs: `message.content.draftPost` (WIX JSON) and `newsLetterSummary` (HTML string)
   - Keep **temperature ≤ 0.3** for stable formatting

3. **Generate an image (OpenAI)**
   - Prompt references the post title and weekly theme
   - Outputs **binary**

4. **Code – set binary filename**
   - Ensure `binary.data.fileName = 'upload.png'`

5. **HTTP Request – Generate Upload URL (Wix Media)**
   - Method: `POST`
   - URL: `/site-media/v1/files/generate-upload-url`
   - Use your Wix credential

6. **HTTP Request – Upload (PUT to uploadUrl)**
   - Method: `PUT`
   - URL: `{{$json["uploadUrl"]}}?filename=upload.png`
   - Send the image binary from previous node

7. **MergeEN (Code) – Inject Image URL into WIX JSON**
   - Set: `message.content.draftPost.richContent.nodes[0].imageData.image.src.id = $json["file"]["url"]`
   - This sets the **hero image** on the first IMAGE node

8. **WIX Post EN (HTTP Request)**
   - URL: `/blog/v3/draft-posts`
   - Body: `{ "draftPost": {{ $json["message"]["content"]["draftPost"] }} }`
   - Language: default (English)

9. **Portuguese translation (chatgpt‑4o‑latest)**
   - Ensures **structure-preserving** translation
   - Output path: `choices[0].message.content.draftPost`

10. **MergeImagePT (Code) – Inject Image URL for PT**
    - Set: `choices[0].message.content.draftPost.richContent.nodes[0].imageData.image.src.id = $json["file"]["url"]`

11. **WIX Post PT (HTTP Request)**
    - URL: `/blog/v3/draft-posts?language=pt`
    - Body: `{ "draftPost": {{ $json["choices"][0]["message"]["content"]["draftPost"] }} }`

12. **Newsletter (chatgpt‑4o‑latest)**
    - Builds HTML using `newsLetterSummary` + hero image URL from WIX JSON
    - Outputs `email_config.subject` and `email_config.html_content`

13. **SendGrid – Create Single Send (HTTP Request)**
    - URL: `/marketing/singlesends`
    - Body includes: `name`, `status: "scheduled"`, `email_config`, and `send_to.list_ids`
    - Replace `list_ids` with your actual SendGrid List ID(s)

---

## 4) First Run (Manual Test)

1. Click **Execute Workflow**.
2. Watch execution:
   - News results fetched
   - Draft Post JSON and newsletter HTML generated
   - Image generated and uploaded
   - EN draft posted to Wix
   - PT draft posted to Wix (language=pt)
   - Newsletter SingleSend created in SendGrid
3. Verify:
   - **WIX** → Blog (Drafts) contains two new drafts (EN, PT)
   - **SendGrid** → Marketing → Single Sends shows a scheduled draft

> If any node fails, open its **Execution Data** to inspect inputs/outputs.

---

## 5) Schedule Weekly Runs

1. Add/enable a **CRON** node at the start of the workflow
2. Suggested: **Every Monday at 08:30 CET** (so content is live by 09:00)
3. Activate the workflow (**toggle ON**)

---

## 6) Troubleshooting

- **No thumbnail image in Wix**: Ensure the **image URL injection** path is correct:  
  `...richContent.nodes[0].imageData.image.src.id`
- **PT draft missing image**: Confirm the PT path is correct:  
  `choices[0].message.content.draftPost.richContent.nodes[0].imageData.image.src.id`
- **SendGrid draft fails**: Check `email_config.subject`, valid HTML, and `send_to.list_ids`.
- **Model formatting drift**: Lower temperature (≤ 0.3) and keep prompts strict about schema.
- **Auth errors**: Reconnect credentials; ensure correct scopes/permissions in Wix and SendGrid.

---

## 7) Security & Production Notes

- Store API keys in **n8n Credentials**, not node parameters.
- Add **JSON Schema validation** (Ajv) before posting to WIX and SendGrid.
- Add an **HTML lint** step before scheduling newsletters.
- Keep a **human-in-the-loop**: route to a Slack/email approval if validation fails.

---

## 8) Quick Reference (Paths & Endpoints)

- **EN draft JSON path**: `message.content.draftPost`
- **PT draft JSON path**: `choices[0].message.content.draftPost`
- **Hero image URL path**: `richContent.nodes[0].imageData.image.src.id`
- **WIX Blog Drafts**: `POST https://www.wixapis.com/blog/v3/draft-posts`
- **WIX Media Upload**: `POST https://www.wixapis.com/site-media/v1/files/generate-upload-url` → `PUT {uploadUrl}?filename=upload.png`
- **SendGrid Single Send**: `POST https://api.sendgrid.com/v3/marketing/singlesends`

---

### Done!

Your workflow is now ready to run weekly and publish crypto news across WIX (EN/PT) and SendGrid.
